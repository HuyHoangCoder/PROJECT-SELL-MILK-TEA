# Bài 14 — Cấu trúc dự án React & luồng khởi động

> **Phần 3 · Frontend với React** — Thời lượng ước tính: **~70 phút**
> ⬅️ Bài trước: [13 — Viết tài liệu API tự động với Swagger](13-swagger-tai-lieu-api.md) · Bài sau: [15 — React Router v6: `createBrowserRouter`, layout lồng nhau](15-routing-v6.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu **Create React App** là gì, `react-scripts` làm gì thay bạn, vì sao dự án không có file webpack nào.
- Phân biệt `public/` với `src/`, biết chính xác React "chui" vào trang HTML ở chỗ nào.
- Đọc vanh vách `yotea-fe/src/index.js` — bóc từng lớp `createRoot` → `StrictMode` → `Provider` → `PersistGate` → `App`.
- Giải thích được vì sao `useEffect` chạy **2 lần** ở chế độ dev, và vì sao **đừng hoảng**.
- Nói được **component** là gì, **JSX** khác HTML ở chỗ nào.
- Dùng lại thành thạo `useState` và `useEffect` (mảng dependency, cleanup).
- Biết mỗi thư mục trong `src/` chứa gì, mỗi package trong `package.json` phục vụ chức năng nào.
- Tự tay tạo component đầu tiên và nhìn thấy nó hiện trên trình duyệt.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 13 — Swagger](13-swagger-tai-lieu-api.md).
- Chạy được frontend ở [Bài 02](02-cai-dat-moi-truong.md): `cd yotea-fe && npm start` → `http://localhost:3000`.
- Đọc lại [Bài 03](03-kien-thuc-nen.md) mục destructuring, arrow function, spread/rest — bài này dùng liên tục.

> 🧵 **Mạch thực hành xuyên suốt.** Ở bài trước bạn đã viết xong **tài liệu Swagger cho API Topping** —
> tới đây backend của chức năng Topping (model, route, controller, slug, bộ lọc query, phân quyền, tài liệu)
> coi như **hoàn tất 100%**. Bài này ta **bước sang frontend**: trước khi dựng được giao diện Topping,
> bạn phải biết dự án React này lắp ráp thế nào và chạy từ đâu tới đâu. Từ [Bài 15](15-routing-v6.md)
> trở đi, mỗi bài thêm đúng một mảnh cho màn hình Topping.

---

## 1. Create React App và `react-scripts`

Trình duyệt **không hiểu** JSX, cũng không xử lý nổi hàng trăm file `import` một cách hiệu quả. Nếu tự
làm, bạn phải cài và cấu hình: **Babel** (dịch JSX → JS thường), **webpack** (gom file thành bundle),
**dev server** (chạy local, tự reload), **PostCSS** (chỗ Tailwind chen vào), **ESLint**.

**Create React App** gói tất cả vào **một package duy nhất**: `react-scripts`. Đó là lý do trong
`yotea-fe/` bạn không thấy `webpack.config.js` — nó nằm bên trong `node_modules/react-scripts`.

`yotea-fe/package.json:36-42`

```json
  "scripts": {
    
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
```

| Lệnh bạn gõ | Thực chất chạy | Kết quả |
|---|---|---|
| `npm start` | `react-scripts start` | Dev server cổng **3000**, tự reload khi lưu file |
| `npm run build` | `react-scripts build` | Sinh `build/` đã nén để deploy ([Bài 36](36-build-va-deploy.md)) |
| `npm test` | `react-scripts test` | Chạy test bằng Jest |
| `npm run eject` | `react-scripts eject` | **Một chiều, không quay lại được** — bung ~30 file cấu hình ra ngoài |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 37 là một **dòng trống** nằm giữa object `scripts`. JSON
> vẫn hợp lệ nên không ai báo lỗi, nhưng đó là rác định dạng — dấu hiệu file này chưa từng được dọn.

> ⚠️ **Đừng bao giờ gõ `npm run eject`** khi đang học.

**CRA còn một "phép màu" nữa: Tailwind.** Dự án **không có** `postcss.config.js`, `devDependencies`
chỉ có mỗi `tailwindcss` (`package.json:61-63`). Tailwind vẫn chạy vì `react-scripts@5` **tự dò tìm**
`tailwind.config.js` ở gốc app, thấy có là tự nhét plugin vào chuỗi PostCSS.

> 💡 Nếu sau này bạn dựng dự án bằng **Vite**, phép màu này **không có** — phải tự viết
> `postcss.config.js` và tự cài `autoprefixer`.

---

## 2. `public/` và `src/` — hai thế giới khác nhau

| | `public/` | `src/` |
|---|---|---|
| Chứa gì | File **tĩnh**: `index.html`, `favicon.ico`, `logo192.png`, `manifest.json`, `robots.txt` | Toàn bộ **code**: `.js`, `.css` |
| Bị Babel/webpack đụng vào? | **Không** — copy nguyên xi sang `build/` | **Có** — bị dịch, gom, nén |
| Tham chiếu trong HTML | `%PUBLIC_URL%/ten-file` | Dùng `import` |
| Sửa file ở đây thì | Phải **Ctrl + F5** | Tự reload ngay |

### 2.1. `public/index.html` — cái khung trống

`yotea-fe/public/index.html:29-31`

```html
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
```

Đọc kỹ: **cả website Yotea — 57 route, hàng chục trang — chỉ có đúng một `<div id="root">` rỗng.**
Đó chính là định nghĩa của **SPA (Single Page Application)**:

```
Website truyền thống (PHP, JSP…)          SPA (React)
──────────────────────────────            ─────────────────────────────
/trang-chu → server trả trang-chu.html    Server trả DUY NHẤT index.html
/san-pham  → server trả san-pham.html     rồi JavaScript vẽ lại nội dung
/lien-he   → server trả lien-he.html      bên trong <div id="root">
  ↑ mỗi lần bấm là TẢI LẠI cả trang          ↑ không tải lại, chỉ đổi ruột
```

`<noscript>` ở dòng 30 là câu an ủi cho ai tắt JavaScript — họ sẽ chỉ thấy đúng dòng chữ đó.

### 2.2. Một chi tiết nhỏ nhưng sai

`yotea-fe/public/index.html:27`

```html
    <title>React App</title>
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** cả `<title>` lẫn `<meta name="description">` (`index.html:8-11`,
> nội dung *"Web site created using create-react-app"*) vẫn để **mặc định của CRA**. Khoảnh khắc đầu
> tiên tải trang, tab trình duyệt hiện chữ **"React App"**, chỉ đổi khi một page nào đó gọi hàm
> `updateTitle` — `yotea-fe/src/utils/index.js:42-44`:
>
> ```js
> export const updateTitle = (title) => {
>   document.title = `${title} - Trà sữa Yotea`;
> };
> ```
>
> **Cách đúng:** sửa thẳng `<title>` trong `public/index.html`, và dùng `react-helmet` để đổi
> title/meta theo từng trang.

---

## 3. Soi code thật: luồng khởi động trong `src/index.js`

Đây là **file đầu tiên** được chạy trong toàn bộ frontend. 21 dòng, mỗi dòng là một mắt xích.

`yotea-fe/src/index.js:1-21`

```js
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import App from "./App";
import reportWebVitals from "./reportWebVitals";
import { Provider } from "react-redux";
import { PersistGate } from "redux-persist/integration/react";
import persistor, { store } from "./redux/store";

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

reportWebVitals();
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import React from "react"` | Thư viện lõi. Cần vì dòng 12 dùng `React.StrictMode` |
| 2 | `import ReactDOM from "react-dom/client"` | Bản **React 18**. Chú ý đuôi `/client` — bản 17 trở về trước là `"react-dom"` |
| 3 | `import "./index.css"` | Nạp CSS toàn cục. Import CSS không gán vào biến nào |
| 4 | `import App from "./App"` | Component gốc, chứa toàn bộ bảng route |
| 5 | `import reportWebVitals ...` | Công cụ đo hiệu năng có sẵn của CRA |
| 6 | `import { Provider } from "react-redux"` | Cầu nối bơm Redux store vào cây React |
| 7 | `import { PersistGate } ...` | Cổng chặn: chờ đọc xong localStorage rồi mới cho vẽ |
| 8 | `import persistor, { store } from "./redux/store"` | **Bẫy!** `persistor` là **default export**, `store` là **named export** |
| 10 | `ReactDOM.createRoot(document.getElementById("root"))` | Tìm `<div id="root">` ở `public/index.html:31` và "cắm rễ" React vào |
| 11 | `root.render(...)` | Ra lệnh vẽ cây component |
| 12 | `<React.StrictMode>` | Chế độ nghiêm ngặt — chỉ có tác dụng ở dev |
| 13 | `<Provider store={store}>` | Từ đây trở xuống dùng được `useSelector` / `useDispatch` |
| 14 | `<PersistGate loading={null} ...>` | `loading={null}` = trong lúc chờ **không vẽ gì** (màn hình trắng) |
| 15 | `<App />` | Bắt đầu ứng dụng thật |
| 21 | `reportWebVitals()` | **Gọi mà không truyền callback** → code chết, xem mục 8.4 |

> ⚠️ **Chỗ này dự án làm chưa chuẩn (dòng 8):** `src/redux/store.js:34` viết
> `export default persistStore(store);` — nghĩa là **default export KHÔNG phải store**, mà là
> **persistor**. Ai quen tay gõ `import store from "./redux/store"` sẽ nhận về persistor và app vỡ
> ngay mà không hiểu vì sao. Rõ ràng hơn nên là `export const persistor = persistStore(store);`.

### 3.1. Bóc từng lớp hành

```
Trình duyệt tải  public/index.html  → gặp <div id="root"></div> (rỗng)
        ▼
src/index.js:10   ReactDOM.createRoot(...)   → React nhận quyền quản lý div đó
        ▼
┌─ <React.StrictMode>          (index.js:12) — chỉ ở DEV: render 2 lần để tìm bug
│ ┌─ <Provider store={store}>  (index.js:13) — mọi con gọi được useSelector/useDispatch
│ │ ┌─ <PersistGate ...>       (index.js:14) — đọc localStorage key "persist:root",
│ │ │                                          dispatch persist/REHYDRATE, CHỜ xong mới vẽ
│ │ │ ┌─ <App />               (App.js:58)
│ │ │ │    ├── createBrowserRouter([...])        (App.js:59-350)
│ │ │ │    ├── <RouterProvider router={router} /> (App.js:354)
│ │ │ │    └── <ToastContainer />                 (App.js:356)
│ │ │ └──────────────────────────────────────────────────────
│ │ └────────────────────────────────────────────────────────
│ └──────────────────────────────────────────────────────────
└────────────────────────────────────────────────────────────
        ▼
WebsiteLayout / AdminLayout / MyAccountLayout → <Outlet /> → trang cụ thể
```

**Thứ tự các lớp là bắt buộc, không đảo được:**

| Câu hỏi | Trả lời |
|---|---|
| Vì sao `PersistGate` phải nằm **trong** `Provider`? | Nó cần `dispatch` action `persist/REHYDRATE` vào store. Không có `Provider` bọc ngoài thì không có store để dispatch |
| Vì sao `App` nằm trong `PersistGate`? | Để khi trang bắt đầu vẽ, thông tin đăng nhập và giỏ hàng đã nạp lại xong. Nếu không, người vừa F5 sẽ thấy mình "bị đăng xuất" trong tích tắc |
| `loading={null}` nghĩa là gì? | Trong lúc chờ, **không vẽ gì**. Đáng lẽ nên truyền `loading={<Loading active />}` để có spinner |

`yotea-fe/src/App.js:352-358`

```jsx
  return (
    <>
      <RouterProvider router={router} />

      <ToastContainer />
    </>
  );
```

`<ToastContainer />` là "khay" hiển thị thông báo nổi, đặt **ngoài** router để trang nào gọi
`toast.success("...")` cũng hiện ra được. Chi tiết router là [Bài 15](15-routing-v6.md).

---

## 4. `StrictMode` và chuyện `useEffect` chạy 2 lần

Đây là câu hỏi số 1 của người mới học React 18. Bạn viết mảng dependency `[]` — nghĩa là "chỉ chạy
**một lần**" — nhưng Console lại in ra **hai** dòng. **Bạn không sai. React cố tình làm thế.**

Từ React 18, khi bọc `<React.StrictMode>` và chạy ở **development**, React sẽ:

1. Mount component (chạy effect lần 1) → 2. **Unmount ngay** (chạy cleanup) → 3. **Mount lại** (chạy effect lần 2)

Mục đích: **bắt lỗi effect không dọn dẹp sau mình**. Nếu effect đăng ký listener, mở `setInterval`
hay WebSocket mà không đóng lại, sau vòng lặp giả này bạn sẽ có **2 cái** đang chạy — bug lộ ra ngay
tại máy bạn thay vì ở máy người dùng.

| Điều phải nhớ | Chi tiết |
|---|---|
| ✅ **Chỉ xảy ra ở dev** | `npm run build` rồi deploy thì effect chạy **đúng 1 lần**. Đừng vì sợ mà xoá `StrictMode` |
| ✅ **Đừng "chữa" bằng cách xoá StrictMode** | Xoá đi là giấu bệnh, không phải chữa bệnh |
| ⚠️ **Nhưng cẩn thận với side-effect thật** | Nếu effect gọi API `POST` tạo đơn hàng, bạn sẽ tạo **2 đơn**. Effect chỉ nên `GET`, hoặc phải chống trùng |

### 4.1. Một effect trong dự án đang dính đúng bẫy này

`yotea-fe/src/pages/layouts/WebsiteLayout.js:55-67`

```js
  useEffect(() => {
    window.addEventListener("scroll", () => {
      const scrollTop = window.scrollY;
      setVisible(scrollTop > 1000 ? true : false);
      setHeaderFixed(scrollTop > 1000 ? true : false);
    });

    dispatch(getCates());

    if (isLogged) {
      dispatch(getWishlist(user._id));
    }
  }, []);
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** effect gắn `scroll` listener nhưng **không có cleanup** để
> `removeEventListener`. Hậu quả: ở dev với `StrictMode`, listener gắn **2 lần** → mỗi lần cuộn chạy
> 2 lần logic, và `dispatch(getCates())` bắn **2 request** lên backend. Tệ hơn, callback là arrow
> function **ẩn danh** nên kể cả muốn gỡ cũng không có tham chiếu để gỡ.
>
> **Cách đúng** (code bạn tự viết thêm, dự án chưa có):
>
> ```js
> useEffect(() => {
>   const handleScroll = () => {
>     const scrollTop = window.scrollY;
>     setVisible(scrollTop > 1000);
>     setHeaderFixed(scrollTop > 1000);
>   };
>   window.addEventListener("scroll", handleScroll);
>
>   return () => window.removeEventListener("scroll", handleScroll);  // ← CLEANUP
> }, []);
> ```
>
> Đã kiểm chứng: **trong toàn bộ `yotea-fe/src/` không có một hàm cleanup nào.**

---

## 5. Component và JSX

### 5.1. Component = một hàm trả về giao diện

Quy ước: **tên component phải viết hoa chữ cái đầu.** React dùng chính quy ước này để phân biệt
`<div>` (thẻ HTML thật) với `<Loading />` (component của bạn).

Ví dụ ngắn nhất trong dự án — `yotea-fe/src/components/Loading.js:1-21`:

```jsx
import { PuffLoader } from "react-spinners";

const Loading = ({ active }) => {
  return (
    <div
      id="loading"
      className={`${
        active && "active"
      } z-20 invisible fixed top-0 right-0 bottom-0 left-0`}
    >
      <div
        id="loading__overlay"
        className="transition-all absolute w-full h-full bg-[rgba(0,0,0,0.6)] flex items-center justify-center opacity-0"
      >
        <PuffLoader color="white" />
      </div>
    </div>
  );
};

export default Loading;
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import { PuffLoader } from "react-spinners"` | Mượn component spinner của thư viện khác |
| 3 | `const Loading = ({ active }) => {` | Component nhận **props**, `{ active }` là destructuring |
| 7-9 | `className={\`${active && "active"} …\`}` | Nối chuỗi class động |
| 15 | `<PuffLoader color="white" />` | Dùng component khác như thẻ HTML, truyền prop `color` |
| 21 | `export default Loading` | Xuất ra để file khác import |

> ⚠️ **Chỗ này dự án làm chưa chuẩn (dòng 8):** `${active && "active"}` — khi `active` là `false`,
> biểu thức trả về `false`, template literal ép nó thành **chuỗi `"false"`** ⇒ DOM có class rác
> `class="false z-20 invisible …"`. Nếu `active` là `undefined` thì ra class `"undefined"`.
> **Cách đúng:** `${active ? "active" : ""}`. Anti-pattern này lặp ở `WebsiteLayout.js:227`, `:517`, `:527`.

### 5.2. JSX khác HTML ở chỗ nào?

JSX **trông giống** HTML nhưng thực chất là **JavaScript** — Babel dịch `<div>Xin chào</div>` thành
`React.createElement("div", null, "Xin chào")`. Vì là JavaScript nên nó có luật riêng:

| # | Luật | HTML | JSX |
|---|---|---|---|
| 1 | Thuộc tính class | `class="btn"` | **`className="btn"`** (vì `class` là từ khoá JS) |
| 2 | Thuộc tính for | `for="email"` | `htmlFor="email"` |
| 3 | Tên thuộc tính | `onclick`, `tabindex` | **camelCase**: `onClick`, `tabIndex` |
| 4 | Thẻ tự đóng | `<img>`, `<br>` hợp lệ | **Bắt buộc** `<img />`, `<br />` |
| 5 | Nhúng biểu thức | Không có | **`{bienJS}`** — dấu ngoặc nhọn |
| 6 | Style inline | `style="color: red"` (chuỗi) | `style={{ color: "red" }}` (object) |
| 7 | Gốc | Nhiều thẻ ngang hàng cũng được | **Chỉ được 1 thẻ cha duy nhất** |
| 8 | Comment | `<!-- … -->` | `{/* … */}` |

**Luật 5 — dấu `{}` nhúng biểu thức.** Sáu dòng dưới đây đều là code có thật trong
`yotea-fe/src/components/user/ProductContent.js`:

```jsx
{item.name}                          // in ra biến
{formatCurrency(item.price)}          // gọi hàm
{item.categoryId?.name}               // optional chaining
{products?.map((item) => <div />)}    // lặp mảng
{emptyProduct ? <A /> : <B />}        // rẽ nhánh bằng toán tử 3 ngôi
{user && <div>Xin chào</div>}         // hiển thị có điều kiện
```

> 💡 Trong `{}` chỉ đặt được **biểu thức** (thứ trả ra giá trị), **không** đặt được `if`, `for`,
> `switch`. Đó là lý do JSX phải dùng `? :` và `&&` thay cho `if/else`.

**Luật 7 — phải có một thẻ cha.**

```jsx
// ❌ SAI — hai thẻ ngang hàng
return (
  <h1>Tiêu đề</h1>
  <p>Nội dung</p>
);

// ✅ ĐÚNG cách 1 — bọc <div> (nhưng sinh thêm 1 thẻ thừa trong DOM)
return (<div><h1>Tiêu đề</h1><p>Nội dung</p></div>);

// ✅ ĐÚNG cách 2 — Fragment <> </> (KHÔNG sinh thẻ nào trong DOM)
return (<><h1>Tiêu đề</h1><p>Nội dung</p></>);
```

Dự án dùng cách 2 rất nhiều — `yotea-fe/src/pages/user/HomePage.js:16-26`:

```jsx
  return (
    <>
      <HomeBanner />
      <HomeCategory />
      <HomeWhy />
      <HomeProducts />
      <HomeNews />
      <HomeFeedback />
      <HomeShow />
    </>
  );
```

Trang chủ chỉ là **7 component ghép lại**. Đó là sức mạnh của tư duy component: chia màn hình lớn
thành mảnh nhỏ, mỗi mảnh một file, sửa mảnh nào chỉ mở đúng file đó.

---

## 6. Ôn nhanh hai hook cốt lõi

> 📖 **Thuật ngữ:** *hook* — hàm bắt đầu bằng `use`, cho phép component "móc" vào tính năng của React.
> **Luật vàng:** hook chỉ được gọi ở **cấp cao nhất** của component — không đặt trong `if`, `for`,
> hay hàm lồng bên trong.

### 6.1. `useState` — bộ nhớ của component

```js
const [giaTri, setGiaTri] = useState(giaTriKhoiTao);
```

Nó trả về **mảng 2 phần tử**: `giaTri` (chỉ để đọc) và `setGiaTri` (gọi hàm này → React **vẽ lại**
component).

Ví dụ thật — `yotea-fe/src/components/user/ProductContent.js:17-24`:

```js
  const [emptyProduct, setEmptyProduct] = useState(false);
  const [totalProduct, setTotalProduct] = useState(0);
  const [products, setProducts] = useState([]);
  const [filter, setFilter] = useState({
    view: "grid",
    sort: "createdAt",
    order: "desc",
  });
```

| Dòng | State | Khởi tạo | Dùng để |
|---|---|---|---|
| 17 | `emptyProduct` | `false` | Cờ báo "không tìm thấy sản phẩm nào" |
| 18 | `totalProduct` | `0` | Tổng số sản phẩm, để tính số trang |
| 19 | `products` | `[]` | Danh sách sản phẩm của trang hiện tại |
| 20-24 | `filter` | object 3 trường | Kiểu xem (lưới/danh sách), trường sắp xếp, chiều sắp xếp |

> ⚠️ **Tuyệt đối không gán trực tiếp:** `products.push(x)` hay `filter.view = "list"` sẽ **không** làm
> React vẽ lại, vì React so sánh **tham chiếu** object. Luôn tạo giá trị **mới**:
> `setProducts([...products, x])`, `setFilter({ ...filter, view: "list" })`.

Đổi state trong dự án — `ProductContent.js:70-72`:

```js
  const handlerUpdateFilter = (data) => {
    setFilter(data);
  };
```

Hàm này được truyền xuống component con qua prop `onUpdateFilter` (`ProductContent.js:136`) — mẫu
**"con báo lên cha"** kinh điển: state ở cha, con chỉ gọi hàm cha đưa cho.

### 6.2. `useEffect` — chạy việc phụ sau khi vẽ xong

```js
useEffect(() => {
  // thân: chạy SAU khi component vẽ xong
  return () => {
    // cleanup: chạy TRƯỚC lần chạy kế tiếp, và khi component bị gỡ khỏi màn hình
  };
}, [mangDependency]);
```

**Mảng dependency quyết định TẤT CẢ:**

| Viết | Effect chạy khi nào | Dùng khi |
|---|---|---|
| `useEffect(fn)` — **không có mảng** | Sau **mỗi lần** render → rất dễ vòng lặp vô tận | Gần như không bao giờ |
| `useEffect(fn, [])` — **mảng rỗng** | **Một lần** khi component xuất hiện (dev: 2 lần vì StrictMode) | Tải dữ liệu ban đầu, đặt tiêu đề trang |
| `useEffect(fn, [x, y])` | Lần đầu **+ mỗi khi `x` hoặc `y` đổi** | Tải lại dữ liệu khi đổi trang / đổi bộ lọc |

**Ví dụ `[]`** — `yotea-fe/src/pages/user/HomePage.js:11-14`:

```js
const HomePage = () => {
  useEffect(() => {
    updateTitle("Trang chủ");
  }, []);
```

Đặt tiêu đề tab thành *"Trang chủ - Trà sữa Yotea"*, đúng một lần khi vào trang. Gần như **mọi page**
trong dự án đều mở đầu bằng đoạn này.

**Ví dụ `[x, y, z]`** — `yotea-fe/src/components/user/ProductContent.js:56-68`:

```js
    const getTotalProduct = async () => {
      const { data } = await getProducts(
        0,
        0,
        filter.sort,
        filter.order,
        parameter
      );
      setEmptyProduct(!data.length ? true : false);
      setTotalProduct(data.length);
    };
    getTotalProduct();
  }, [page, parameter, filter]);
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 56 | `const getTotalProduct = async () => {` | Định nghĩa hàm async **bên trong** effect |
| 57-63 | `await getProducts(0, 0, …)` | Gọi API lấy **toàn bộ** sản phẩm (start=0, limit=0) |
| 64 | `setEmptyProduct(!data.length ? true : false)` | Mảng rỗng → bật cờ "không có sản phẩm" |
| 65 | `setTotalProduct(data.length)` | Lưu tổng số, dùng tính `totalPage` ở dòng 27 |
| 67 | `getTotalProduct()` | **Gọi** hàm vừa định nghĩa |
| 68 | `}, [page, parameter, filter])` | Chạy lại mỗi khi **đổi trang**, **đổi danh mục**, hoặc **đổi bộ lọc** |

> 💡 **Vì sao phải định nghĩa hàm async bên trong rồi mới gọi?** Vì callback của `useEffect`
> **không được là `async`**. Viết `useEffect(async () => {…}, [])` thì hàm trả về một Promise, mà React
> lại tưởng giá trị trả về đó là hàm cleanup → cảnh báo và lỗi khó hiểu.

### 6.3. Cleanup function

Cleanup chạy **hai lúc**: trước mỗi lần effect chạy lại, và khi component biến mất khỏi màn hình.

```js
// đoạn này bạn tự viết thêm để hiểu cơ chế, dự án chưa có
useEffect(() => {
  const id = setInterval(() => console.log("tick"), 1000);
  return () => clearInterval(id);   // ← không có dòng này thì đồng hồ chạy mãi
}, []);
```

| Việc effect làm | Cleanup phải làm |
|---|---|
| `addEventListener` | `removeEventListener` |
| `setInterval` / `setTimeout` | `clearInterval` / `clearTimeout` |
| Mở WebSocket / kết nối | Đóng lại |
| Gọi API | `abortController.abort()` để huỷ request cũ |

---

## 7. Bản đồ thư mục `src/`

```
yotea-fe/src/
├── api/                 ← 15 file: tầng gọi HTTP tới backend
├── components/          ← mảnh giao diện tái sử dụng
│   ├── admin/           ← 13 file: bảng danh sách, phân trang, PrivateRouter…
│   ├── user/            ← 14 file + thư mục home/ (7 khối trang chủ)
│   └── Loading.js       ← dùng chung cả 2 phía
├── pages/               ← MỖI FILE = MỘT MÀN HÌNH gắn với 1 route
│   ├── admin/           ← cart, category, category-news, contact, news,
│   │                       product, profile, slider, user + Dashboard.js
│   ├── auth/            ← LoginPage, RegisterPage, ForgotPage
│   ├── layouts/         ← WebsiteLayout, AdminLayout, MyAccountLayout
│   └── user/            ← HomePage, ProductPage… + cart/ + my-account/
├── redux/               ← 13 file: store, rootReducer + 11 slice
├── utils/               ← index.js (hàm tiện ích), localStorage.js
├── App.js               ← component gốc, chứa toàn bộ bảng route
├── App.css              ← ⚠️ CODE CHẾT
├── App.test.js          ← ⚠️ test mặc định CRA, chạy là FAIL
├── index.js             ← điểm khởi động
├── index.css            ← CSS toàn cục + 3 directive Tailwind
├── logo.svg             ← ⚠️ không dùng
├── reportWebVitals.js   ← ⚠️ có gọi nhưng không làm gì
└── setupTests.js        ← cấu hình jest-dom
```

| Thư mục / file | Chứa gì | Bài liên quan |
|---|---|---|
| `api/` | Mỗi file gói 1 nhóm endpoint theo quy ước `getAll`/`get`/`add`/`update`/`remove`. `instance.js` là axios instance dùng chung | [18](18-tang-api-axios.md), [22](22-rtk-query.md) |
| `components/admin/` | Mảnh giao diện quản trị: `UserList`, `ListProduct`, `AdminPagination`, `PrivateRouter`… | [24](24-private-router.md), [32](32-trang-quan-tri.md) |
| `components/user/` | Mảnh giao diện khách: `ProductContent`, `FilterProduct`, `Pagination`, `CommentList`… | [16](16-layout-va-component.md), [25](25-danh-sach-san-pham.md) |
| `components/user/home/` | 7 khối trang chủ: `HomeBanner`, `HomeCategory`, `HomeProducts`… | [31](31-tin-tuc-lien-he-cua-hang.md) |
| `pages/layouts/` | Khung bao ngoài: header + `<Outlet />` + footer. 3 layout cho 3 khu vực | [15](15-routing-v6.md), [16](16-layout-va-component.md) |
| `pages/auth/` | 3 màn hình đăng nhập / đăng ký / quên mật khẩu | [23](23-dang-ky-dang-nhap.md) |
| `pages/user/` | Màn hình khách, có 2 thư mục con `cart/` (3 file) và `my-account/` (4 file) | [25](25-danh-sach-san-pham.md)–[30](30-binh-luan-danh-gia-yeu-thich.md) |
| `pages/admin/` | Màn hình quản trị chia theo nghiệp vụ, mỗi nghiệp vụ có `List`/`Add`/`Edit` | [32](32-trang-quan-tri.md) |
| `redux/` | `store.js` (store + persist), `rootReducer.js` (gộp reducer), 11 slice trạng thái | [19](19-redux-toolkit-co-ban.md)–[21](21-redux-persist.md) |
| `utils/` | `index.js`: `uploadFile`, `formatCurrency`, `formatDate`, `formatDateNews`, `updateTitle`. `localStorage.js`: `isAuthenticate()` | [33](33-ra-soat-bao-mat.md) |
| `index.css` | 291 dòng: 3 directive Tailwind + CSS thuần | [17](17-tailwind-css.md) |
| `App.css` | Nguyên bản template CRA (`.App-logo` xoay tròn) — **không file nào import** | — |
| `reportWebVitals.js` | Hàm đo hiệu năng của CRA | mục 8.4 |
| `setupTests.js` | Nạp matcher của `jest-dom` cho Jest | — |

> 💡 **Quy tắc phân biệt:** thứ nào **gắn với một URL** thì vào `pages/`; thứ nào **được nhiều nơi
> dùng lại** thì vào `components/`. Ranh giới này bạn phải tự giữ — công cụ không ép được.

### 7.1. Mỗi package trong `package.json` để làm gì

`yotea-fe/package.json:5-35`

| Package | Vai trò **thực tế** trong Yotea |
|---|---|
| `react` · `react-dom` | React 18. `react-dom/client` cho `createRoot` (`index.js:2`, `:10`) |
| `react-scripts` | CRA 5 — dev server, build, tự bật Tailwind |
| `react-router-dom` | `createBrowserRouter` + `RouterProvider` (Data Router API v6.4+), 57 route trong `App.js` |
| `@reduxjs/toolkit` | `createSlice`, `createAsyncThunk`, `combineReducers`, **và RTK Query** (`createApi`, `fetchBaseQuery`) |
| `react-redux` | `Provider` (`index.js:13`), `useSelector`, `useDispatch` |
| `redux` | ⚠️ Dùng trực tiếp `createStore` + `applyMiddleware` (`store.js:4`, `:30-33`) — lối cũ |
| `redux-thunk` | Middleware thunk. Bản v3 export **named**: `import { thunk }` (`store.js:5`) |
| `redux-persist` | `persistReducer`, `persistStore`, `PersistGate` — lưu `auth` và `cart` xuống localStorage |
| `axios` | HTTP client cho toàn bộ `src/api/*.js` + upload ảnh lên Cloudinary |
| `react-hook-form` | `useForm`, `register`, `handleSubmit` — mọi form thêm/sửa |
| `yup` | Viết schema kiểm tra dữ liệu form ("email bắt buộc", "mật khẩu ≥ 6 ký tự") |
| `@hookform/resolvers` | Cầu nối `react-hook-form` ↔ `yup` qua `yupResolver` |
| `tailwindcss` | Toàn bộ CSS viết bằng class Tailwind. Là **devDependency** |
| `react-toastify` | Thông báo nổi góc màn hình. `<ToastContainer />` ở `App.js:356`, CSS ở `App.js:26` |
| `sweetalert2` | Hộp thoại xác nhận "Bạn chắc chắn muốn xoá?" ở màn admin. CSS ở `App.js:25` |
| `react-slick` | Slider/carousel: `HomeBanner`, `HomeFeedback`, `HomeShow` |
| `slick-carousel` | CSS gốc của slick (`App.js:27-28`) — **bắt buộc đi kèm** `react-slick` |
| `react-spinners` | `<PuffLoader />` trong `components/Loading.js:15` — chỉ dùng đúng 1 chỗ |
| `@fortawesome/fontawesome-svg-core` | Lõi FontAwesome, bắt buộc kèm bộ icon |
| `@fortawesome/react-fontawesome` | Component `<FontAwesomeIcon icon={…} />` |
| `@fortawesome/free-solid-svg-icons` | Bộ icon chính: `faBars`, `faSearch`, `faShoppingCart`, `faHeart`, `faTrash`… |
| `@fortawesome/free-brands-svg-icons` | Icon mạng xã hội ở footer: `faFacebookF`, `faYoutube`, `faInstagram`, `faTiktok` |
| `uuid` | Sinh id duy nhất cho từng món trong giỏ hàng — chỉ dùng ở `ProductDetailPage.js` |
| `web-vitals` | Đo hiệu năng, chỉ dùng trong `reportWebVitals.js` |
| `@testing-library/*` | Bộ test có sẵn của CRA |

---

## 8. ⚠️ Bốn vấn đề hạ tầng cần biết ngay từ bây giờ

### 8.1. `App.css` là code chết 100%

`yotea-fe/src/App.css:1-8`

```css
.App {
  text-align: center;
}

.App-logo {
  height: 40vmin;
  pointer-events: none;
}
```

> ⚠️ Nguyên xi template CRA (logo React xoay tròn). Đã kiểm chứng bằng tìm kiếm toàn bộ `src/`:
> **không file nào import `App.css`**. Cùng nhóm code chết còn có `src/logo.svg`, `src/App.test.js`,
> `src/setupTests.js`. Riêng `App.test.js:4-8` khiến `npm test` **FAIL** ngay vì nó đi tìm chữ
> "learn react" — thứ đã bị xoá từ đời nào. **Cách đúng:** xoá các file này hoặc viết test thật
> ([Bài 34](34-refactor-du-an.md)).

### 8.2. Hai dependency thừa

| Package | Vấn đề |
|---|---|
| `slugify` (`package.json:30`) | **Không được import ở bất kỳ file nào trong `src/`.** Slug đã sinh ở backend ([Bài 08](08-slug-slugify.md)), frontend chỉ nhận về dùng |
| `@fortawesome/free-regular-svg-icons` (`package.json:8`) | **Không được import ở đâu.** Dự án chỉ dùng bộ `solid` và `brands` |

> 💡 Dependency thừa không gây lỗi chạy, nhưng làm `node_modules` phình to và tăng bề mặt rủi ro
> bảo mật. Mẹo kiểm tra: chạy `npx depcheck` trong `yotea-fe/`.

### 8.3. `index.css` ẩn **toàn bộ** iframe của website

`yotea-fe/src/index.css:1-11`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

iframe {
  display: none;
}

.store__list-map > div > iframe {
  @apply w-full h-full rounded-lg;
}
```

> ⚠️ **Rất nguy hiểm.** Dòng 5-7 là selector **toàn cục**: mọi `<iframe>` trên toàn site đều bị ẩn —
> Google Maps, video YouTube nhúng, plugin Facebook, cổng thanh toán… Dòng 9-11 là miếng **vá** để
> bật lại đúng cái bản đồ ở trang Cửa hàng.
>
> Hậu quả thực tế: mai bạn nhúng một video vào trang chủ, nó sẽ **vô hình** và bạn mất cả buổi để
> hiểu vì sao. **Cách đúng:** nhắm đúng selector cần ẩn (`.some-widget iframe { display: none; }`)
> thay vì ẩn tất cả rồi vá lại.

### 8.4. Không có trang 404 — và `reportWebVitals()` không làm gì

> ⚠️ Router trong `App.js:59-350` **không khai `errorElement`, cũng không có route `path: "*"`**.
> Gõ một URL sai (ví dụ `http://localhost:3000/khong-ton-tai`) sẽ ra màn hình lỗi mặc định xấu xí
> **"Unexpected Application Error!"** kèm stack trace — vừa xấu vừa lộ thông tin kỹ thuật.
>
> Đây không phải chuyện lý thuyết: dự án đang có **2 link chết** trỏ tới route không tồn tại —
> `WebsiteLayout.js:298` (`/gio-hang`, route thật là `/cart`) và `AdminLayout.js:209` (`/admin/profile`).
>
> **Cách đúng** (làm ở [Bài 15](15-routing-v6.md) và [Bài 34](34-refactor-du-an.md)):
>
> ```jsx
> // đoạn này bạn tự viết thêm, dự án chưa có
> { path: "*", element: <NotFoundPage /> }
> ```

Bonus — `yotea-fe/src/reportWebVitals.js:1-2`:

```js
const reportWebVitals = onPerfEntry => {
  if (onPerfEntry && onPerfEntry instanceof Function) {
```

`index.js:21` gọi `reportWebVitals()` **không truyền tham số** ⇒ `onPerfEntry` là `undefined` ⇒ điều
kiện dòng 2 sai ⇒ **toàn bộ thân hàm bị bỏ qua**. Muốn nó chạy phải gọi `reportWebVitals(console.log)`.

---

## 9. 🛠️ Tự tay làm — component đầu tiên của bạn

> **Mục tiêu:** cuối phần này bạn có một component tự viết tên `HelloTopping`, hiện chữ trên trang chủ,
> có nút bấm đếm số lần, và in log ra Console khi xuất hiện. Đây là **bước tập đi** trước khi
> [Bài 16](16-layout-va-component.md) bắt bạn viết `ToppingCard` thật.

> 📌 **Toàn bộ code trong phần này là code bạn tự viết thêm — dự án chưa có.**

### Bước 1 — Bật frontend

```bash
cd D:/PROJECT-SELL-MILK-TEA/PROJECT-SELL-MILK-TEA/yotea-fe
npm start
```

Trình duyệt tự mở `http://localhost:3000`. Cứ để terminal đó chạy.

> 💡 Bài này chưa cần bật backend. Trang chủ sẽ báo lỗi gọi API ở Console — kệ nó.

### Bước 2 — Tạo file component

```jsx
// yotea-fe/src/components/user/HelloTopping.js  ← file MỚI, bạn tự tạo
const HelloTopping = () => {
  return (
    <div className="container max-w-6xl mx-auto px-3 py-6 text-center">
      <p className="text-xl font-semibold text-[#D9A953]">
        Xin chào, tôi là component Topping đầu tiên!
      </p>
    </div>
  );
};

export default HelloTopping;
```

| Chi tiết | Vì sao |
|---|---|
| Tên hàm `HelloTopping` **viết hoa chữ đầu** | Không viết hoa, React tưởng là thẻ HTML và không vẽ gì ra |
| `className` chứ không phải `class` | Luật JSX số 1 ở mục 5.2 |
| `export default HelloTopping;` | Không có dòng này thì file khác không import được |

`#D9A953` là **màu thương hiệu** của Yotea ([Bài 17](17-tailwind-css.md) sẽ giải thích vì sao dự án
phải viết kiểu `[#D9A953]` thay vì đặt tên tử tế).

### Bước 3 — Import vào trang chủ

Mở `yotea-fe/src/pages/user/HomePage.js`, thêm **2 dòng**:

```jsx
// yotea-fe/src/pages/user/HomePage.js — dòng import bạn thêm vào
import HelloTopping from "../../components/user/HelloTopping";
```

```jsx
  return (
    <>
      <HelloTopping />        {/* ← dòng bạn thêm */}
      <HomeBanner />
      <HomeCategory />
      ...
```

Lưu file → trình duyệt tự reload → dòng chữ vàng hiện ngay trên banner. 🎉

> 💡 Đường dẫn `"../../components/user/HelloTopping"`: từ `src/pages/user/` lùi ra 2 cấp về `src/`,
> rồi đi vào `components/user/`. Không cần gõ đuôi `.js`.

### Bước 4 — Thêm `useState` đếm số lần bấm

```jsx
// yotea-fe/src/components/user/HelloTopping.js  ← bạn sửa tiếp
import { useState } from "react";

const HelloTopping = () => {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
  };

  return (
    <div className="container max-w-6xl mx-auto px-3 py-6 text-center">
      <p className="text-xl font-semibold text-[#D9A953]">
        Xin chào, tôi là component Topping đầu tiên!
      </p>

      <button
        onClick={handleClick}
        className="mt-3 px-4 py-2 bg-[#D9A953] text-white font-semibold uppercase text-sm"
      >
        Đã bấm {count} lần
      </button>
    </div>
  );
};

export default HelloTopping;
```

| Dòng bạn viết | Ý nghĩa |
|---|---|
| `import { useState } from "react";` | Có ngoặc nhọn — `useState` là named export |
| `const [count, setCount] = useState(0);` | State tên `count`, khởi đầu `0` |
| `setCount(count + 1)` | Đổi state → React vẽ lại component |
| `onClick={handleClick}` | camelCase, truyền **tên hàm**. Thêm cặp ngoặc `handleClick()` là **gọi luôn lúc render** → sai |
| `Đã bấm {count} lần` | Nhúng biến vào JSX bằng `{}` |

### Bước 5 — Thêm `useEffect` in log

```jsx
// yotea-fe/src/components/user/HelloTopping.js  ← bạn thêm tiếp
import { useEffect, useState } from "react";

const HelloTopping = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("HelloTopping đã xuất hiện trên màn hình!");
  }, []);

  // ... phần còn lại giữ nguyên
```

Nhớ sửa dòng import thành `import { useEffect, useState } from "react";`.

---

## 10. ✅ Kiểm chứng kết quả

**Trên trình duyệt.** Mở `http://localhost:3000`, bạn phải thấy **ngay trên banner trang chủ**:

```
        Xin chào, tôi là component Topping đầu tiên!

                 [  ĐÃ BẤM 0 LẦN  ]        ← nút màu vàng #D9A953
```

Bấm 3 lần → chữ đổi thành **"ĐÃ BẤM 3 LẦN"**. Nếu số không đổi, bạn đã viết `onClick={handleClick()}`
(có ngoặc) thay vì `onClick={handleClick}`.

**Trên tab Console.** Nhấn **F12** → tab **Console** → **F5** tải lại trang:

```
HelloTopping đã xuất hiện trên màn hình!
HelloTopping đã xuất hiện trên màn hình!
```

**Hai lần.** Đúng như mục 4: `<React.StrictMode>` ở `index.js:12` cố tình mount–unmount–mount lại ở
dev. **Đây là hành vi bình thường, không phải bug của bạn.**

**Kiểm chứng `<div id="root">`.** Chuyển sang tab **Elements**, tìm `<body>`:

```html
<body>
  <div id="root">
    <!-- toàn bộ website Yotea nằm trong đây -->
  </div>
</body>
```

So với `public/index.html:31` — lúc đầu nó **rỗng hoàn toàn**. Tất cả những gì bạn thấy đều do React
vẽ ra sau khi JavaScript chạy.

**Dọn dẹp.** `HelloTopping` chỉ là bài tập, bạn có thể giữ lại hoặc gỡ 2 dòng đã thêm trong
`HomePage.js`. Từ [Bài 16](16-layout-va-component.md), ta sẽ viết `ToppingCard.js` — component
Topping **thật sự** — và một trang `ToppingPage` riêng thay vì nhét vào trang chủ.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Adjacent JSX elements must be wrapped in an enclosing tag` | Trả về nhiều thẻ ngang hàng | Bọc bằng `<> … </>` hoặc `<div>` |
| `Module not found: Can't resolve '../../components/user/HelloTopping'` | Sai đường dẫn hoặc sai chữ hoa/thường | Đếm lại số `../`; kiểm tra tên file |
| Component không hiện gì, không báo lỗi | Tên component viết **thường** (`helloTopping`) | Viết hoa chữ cái đầu |
| `Warning: Invalid DOM property 'class'. Did you mean 'className'?` | Dùng `class` thay vì `className` | Đổi thành `className` |
| Console in log **2 lần** | `<React.StrictMode>` ở dev | **Không phải lỗi.** Xem mục 4 |
| `Too many re-renders` | Gọi `setState` ngay trong thân component, hoặc `onClick={handleClick()}` có ngoặc | Bỏ cặp ngoặc; đưa `setState` vào `useEffect` hoặc hàm xử lý sự kiện |
| Trang trắng, Console báo `Cannot read properties of null (reading 'auth')` | `isAuthenticate()` ở `utils/localStorage.js:2` khi chưa từng đăng nhập | Đăng nhập một lần, hoặc xem cách vá ở [Bài 33](33-ra-soat-bao-mat.md) |
| `useEffect` chạy vô tận, Network bắn request liên tục | Quên mảng dependency, hoặc để object tạo mới mỗi render vào dependency | Thêm `[]`; đưa object ra ngoài component hoặc dùng `useMemo` |
| Sửa file trong `public/` mà không thấy đổi | `public/` không được hot-reload | **Ctrl + F5** |
| `npm test` FAIL ngay | `src/App.test.js` là test mặc định CRA | Bỏ qua — lỗi có sẵn của dự án (mục 8.1) |

---

## 12. 📝 Bài tập

**Bài 1.** Nếu ai đó **đổi chỗ** `<Provider>` và `<PersistGate>` trong `src/index.js` thành:

```jsx
<PersistGate loading={null} persistor={persistor}>
  <Provider store={store}>
    <App />
  </Provider>
</PersistGate>
```

thì chuyện gì xảy ra? Vì sao?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Ứng dụng vỡ ngay lập tức.**

`PersistGate` cần **dispatch** action `persist/REHYDRATE` để nạp dữ liệu từ localStorage vào store.
Muốn dispatch thì phải có store, mà store được cung cấp qua React Context bởi `<Provider>`. Nếu
`PersistGate` nằm **ngoài** `Provider`, nó ở phía **trên** trong cây component → không "nhìn thấy"
context → lỗi kiểu `Could not find "store" in the context of "Connect(PersistGate)"`.

**Quy tắc chung:** component **cung cấp** context phải bọc **ngoài** component **tiêu thụ** context.
Thứ tự đúng luôn là `StrictMode` → `Provider` → `PersistGate` → `App`.

</details>

**Bài 2.** Sửa `HelloTopping.js` sao cho mỗi khi `count` **thay đổi**, Console in ra
`Số lần bấm hiện tại: <count>`.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Chỉ cần đưa `count` vào mảng dependency (code bạn tự viết thêm):

```jsx
useEffect(() => {
  console.log("Số lần bấm hiện tại:", count);
}, [count]);
```

| Hành động | Console in |
|---|---|
| Tải trang lần đầu | 2 dòng `Số lần bấm hiện tại: 0` (do StrictMode) |
| Bấm nút lần 1 | `Số lần bấm hiện tại: 1` |
| Bấm nút lần 2 | `Số lần bấm hiện tại: 2` |

Lưu ý: effect với `[count]` **vẫn chạy ở lần render đầu tiên**. Mảng dependency không có nghĩa "chỉ
chạy khi đổi", mà là "chạy lần đầu **+** chạy lại khi đổi".

Nếu bạn viết `}, []);` thì Console mãi mãi in `0` — vì effect chụp lại (capture) giá trị `count` tại
thời điểm mount và không bao giờ cập nhật. Đây là bẫy **stale closure** kinh điển.

</details>

**Bài 3.** Với đoạn `useEffect` ở `yotea-fe/src/pages/layouts/WebsiteLayout.js:55-67` (đã trích ở mục
4.1): a) chỉ ra lỗi; b) viết lại cho đúng; c) ở dev có `StrictMode`, hậu quả cụ thể là gì?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**a) Lỗi: thiếu hàm cleanup.** Effect gắn `scroll` listener lên `window` nhưng không bao giờ gỡ ra.
Thêm nữa, callback là arrow function **ẩn danh** nên dù muốn gỡ cũng không có tham chiếu để truyền
vào `removeEventListener`.

**b) Viết lại (code bạn tự viết thêm):**

```js
useEffect(() => {
  const handleScroll = () => {
    const scrollTop = window.scrollY;
    setVisible(scrollTop > 1000);
    setHeaderFixed(scrollTop > 1000);
  };

  window.addEventListener("scroll", handleScroll);
  dispatch(getCates());
  if (isLogged) {
    dispatch(getWishlist(user._id));
  }

  return () => window.removeEventListener("scroll", handleScroll);
}, []);
```

Tiện thể sửa luôn `scrollTop > 1000 ? true : false` thành `scrollTop > 1000` — phép so sánh **đã**
trả về boolean rồi.

**c)** Effect chạy 2 lần → **2 listener** cùng gắn vào `window`. Mỗi lần cuộn chuột logic chạy 2 lần,
và `dispatch(getCates())` bắn **2 request** lên backend. `WebsiteLayout` không bao giờ unmount trong
lúc dùng nên rò rỉ chưa tích luỹ thêm, nhưng nếu đặt đoạn này trong component hay mount/unmount thì
listener chồng chất vô hạn — đúng nghĩa **memory leak**.

</details>

**Bài 4.** Khi bạn gõ `http://localhost:3000/thuc-don` và Enter, **server nào** trả về file gì? Vì sao
dù URL khác nhau, trình duyệt vẫn luôn nhận đúng một file `index.html`?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

- Ở dev, **dev server của `react-scripts`** (cổng 3000) trả về `public/index.html` — **file duy nhất**.
  Backend Express ở cổng 8080 không liên quan; nó chỉ được gọi khi JavaScript bắn request `/api/...`
  bằng axios.
- Dev server được cấu hình **SPA fallback**: mọi đường dẫn không khớp file tĩnh nào đều trả về
  `index.html`. Sau đó React Router đọc `window.location.pathname` = `/thuc-don` rồi quyết định vẽ
  `ProductPage` vào `<div id="root">`.
- **Hệ quả khi deploy** ([Bài 36](36-build-va-deploy.md)): nếu upload `build/` lên web server **không**
  cấu hình fallback (Nginx/Apache mặc định), vào thẳng `/thuc-don` sẽ ra **404 của server** — vì trên
  đĩa không có file nào tên `thuc-don`. Phải thêm `try_files $uri /index.html;` (Nginx) hoặc file
  `_redirects` (Netlify).

</details>

**Bài 5.** Tìm ra **tất cả** file trong `src/` là "code chết" (không được import ở đâu) và giải thích
vì sao chúng còn tồn tại.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

| File | Lý do là code chết |
|---|---|
| `src/App.css` | Template CRA, không file nào import |
| `src/logo.svg` | Logo React của template |
| `src/App.test.js` | Test mặc định tìm chữ "learn react" → chạy là FAIL |
| `src/setupTests.js` | Chỉ phục vụ `App.test.js` |
| `src/reportWebVitals.js` | Có được gọi ở `index.js:21` nhưng **không truyền callback** → thân hàm không chạy |
| `src/pages/admin/profile/AdminUpdateInfoPage.js` | Có code nhưng **không route nào trỏ tới** |
| `src/pages/admin/profile/AdminUpdatePassword.js` | Tương tự |
| `src/api/store.js` + `src/components/admin/StoreList.js` | Có API và component quản lý cửa hàng nhưng **không có route `/admin/store`** |

**Vì sao còn tồn tại?** Bốn file đầu là rác sinh ra lúc `npx create-react-app` — không ai dọn. Nhóm
còn lại là **chức năng làm dở**: code viết xong nhưng quên nối route. Dấu hiệu điển hình của dự án
bài tập chạy nước rút trước deadline.

**Cách tự kiểm chứng:** dùng Find in Files của VS Code (Ctrl+Shift+F), gõ tên file và xem có kết quả
nào ngoài chính file đó không.

</details>

---

## 📌 Tóm tắt

- **Create React App** gói Babel + webpack + dev server + PostCSS vào `react-scripts`. `npm start`
  thực chất là `react-scripts start`. **Đừng bao giờ `eject`.**
- `public/` chứa file tĩnh **không qua xử lý**; `src/` chứa code **bị biên dịch**. Cả website chỉ có
  **một** file HTML với **một** `<div id="root">` rỗng (`public/index.html:31`) — định nghĩa của **SPA**.
- Luồng khởi động: `createRoot` → `StrictMode` → `Provider` (bơm store) → `PersistGate` (chờ đọc
  localStorage) → `App` (router). **Thứ tự bắt buộc**, vì lớp trong cần context của lớp ngoài.
- `<React.StrictMode>` khiến `useEffect` chạy **2 lần** ở **dev** để lộ effect quên cleanup.
  **Không phải bug — đừng xoá StrictMode để "sửa".** Bản build production chạy đúng 1 lần.
- **Component** = hàm trả về JSX, **tên viết hoa chữ đầu**. **JSX** ≠ HTML: `className`, `{}` nhúng
  biểu thức, `style={{}}`, và **phải có 1 thẻ cha** (dùng `<> </>` cho gọn).
- `useState` cho component có **bộ nhớ**; `useEffect` chạy **việc phụ sau khi vẽ**. Dependency:
  `[]` = một lần, `[x]` = lần đầu + mỗi khi `x` đổi, **không có mảng** = mỗi lần render.
- **Cleanup** (`return () => {…}`) bắt buộc khi effect gắn listener/timer. **Cả dự án không có một
  hàm cleanup nào** — điểm yếu bạn phải tránh lặp lại.
- Bốn vấn đề hạ tầng: `App.css` là **code chết**; `slugify` và `free-regular-svg-icons` là
  **dependency thừa**; `index.css:5-7` **ẩn mọi iframe** toàn cục; router **không có `errorElement`
  nên không có trang 404**.

**Từ khoá tra cứu thêm:** `create react app`, `react-scripts`, `ReactDOM.createRoot`, `React.StrictMode double render`, `JSX rules`, `React Fragment`, `useState hook`, `useEffect dependency array`, `useEffect cleanup function`, `single page application`

➡️ **Bài tiếp theo:** [15 — React Router v6: `createBrowserRouter`, layout lồng nhau](15-routing-v6.md) —
bạn đã biết React vẽ vào đâu và vẽ từ đâu; giờ tới lúc hiểu **ai quyết định vẽ trang nào**. Và ngay
trong bài đó, bạn sẽ tự tay thêm route `topping` vào `App.js` để mở đường cho màn hình Topping.
