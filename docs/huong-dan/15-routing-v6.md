# Bài 15 — React Router v6: `createBrowserRouter`, layout lồng nhau

> **Phần 2 · Frontend nền tảng** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [14 — Cấu trúc dự án React & luồng khởi động](14-cau-truc-react-app.md) · Bài sau: [16 — Layout và component tái sử dụng](16-layout-va-component.md) ➡️
> 🏠 [Mục lục](README.md)

---

Ở [Bài 14](14-cau-truc-react-app.md) bạn đã đi hết luồng khởi động frontend: `index.html` → `index.js` → `<Provider>` → `<PersistGate>` → `<App />`. Chuỗi đó dừng đúng ở cửa nhà `App`. **Bài này ta làm tiếp bước ngay sau đó**: khi người dùng gõ `localhost:3000/thuc-don`, ai quyết định vẽ `ProductPage` chứ không phải `HomePage`? Đó là **React Router v6**, và toàn bộ bộ não nằm gọn trong `yotea-fe/src/App.js`.

## 🎯 Sau bài này bạn sẽ

- Giải thích được **SPA** là gì và vì sao bấm menu mà trình duyệt không nạp lại trang.
- Phân biệt dứt khoát `<a href>` với `<Link to>` — và biết vì sao dùng nhầm sẽ **mất state Redux**.
- Đọc hiểu toàn bộ cây route của Yotea, kể cả `path: ""`, `children` và `<Outlet />`.
- Đọc được `:slug`, `:keyword`, `:page`, `:id` bằng `useParams()`; chuyển trang bằng `useNavigate()`.
- Chỉ ra 3 lỗi routing có thật trong dự án và biết cách sửa đúng.
- **Tự tay thêm route `/topping`** — bước đầu tiên dựng giao diện cho API Topping bạn đã viết ở phần backend.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 14](14-cau-truc-react-app.md); backend chạy cổng **8080**, frontend `npm start` cổng **3000**.
- Mở sẵn `yotea-fe/src/App.js` (362 dòng — ta soi nó cả bài).
- Nhớ lại API **Topping** bạn tự xây ở [Bài 04](04-express-va-appjs.md)–[Bài 13](13-swagger-tai-lieu-api.md): `models/topping.js`, `routes/topping.js`, `controllers/topping.js`.

---

## 1. SPA — Single Page Application là gì?

Hình dung hai kiểu quán trà sữa. **Quán kiểu cũ (Multi Page App):** muốn xem menu, bạn phải ra khỏi quán rồi đi vòng vào cửa số 2 — mỗi lần vào lại là bắt đầu từ con số 0, áo khoác treo lại, chỗ ngồi mất. **Quán kiểu mới (SPA):** bạn ngồi yên, nhân viên chỉ **thay tờ menu trên bàn**.

Yotea là quán kiểu mới. Cả website chỉ có **đúng một file HTML** — `yotea-fe/public/index.html` — với một thẻ `<div id="root">` trống. Mọi trang bạn thấy đều là JavaScript vẽ đè vào đó.

Trong Yotea, giỏ hàng nằm ở `state.cart`, đăng nhập ở `state.auth`, yêu thích ở `state.wishlist`. **Redux store sống trong RAM của tab trình duyệt** — tải lại trang là store bị dựng lại từ đầu:

```
Bấm <a href="/cart">  →  gửi request mới lên server, tải lại index.html + toàn bộ bundle JS
                      →  React khởi động lại, Redux store MỚI TINH → trắng màn 1–2 giây

Bấm <Link to="/cart"> →  KHÔNG request nào, chỉ đổi URL bằng History API
                      →  React thay component trong <Outlet />, Redux store GIỮ NGUYÊN
```

| | `<a href="/cart">` | `<Link to="/cart">` |
|---|---|---|
| Thuộc về | HTML thuần | `react-router-dom` |
| Gửi request + tải lại bundle? | **Có** (chậm, nháy trắng) | **Không** |
| Redux store | **Mất, dựng lại** | Giữ nguyên |
| `useState` trong component | **Mất** | Giữ nguyên |
| Vị trí cuộn | Nhảy lên đầu | Giữ nguyên |
| Dùng cho | Link **ra ngoài**: `mailto:`, `tel:`, Facebook | Mọi đường dẫn **nội bộ** |

> ⚠️ Yotea "sống sót" một phần nếu lỡ dùng `<a href>`, vì `auth` và `cart` được `redux-persist` ghi xuống localStorage ([Bài 21](21-redux-persist.md)). Nhưng `wishlist`, `product`, `news`… **không** nằm trong whitelist → mất sạch, phải gọi lại API. Còn `useState` thì mất 100%.

Dự án dùng đúng quy tắc: link nội bộ là `<Link>` (`WebsiteLayout.js:216`), link ra ngoài mới là `<a>`.

`yotea-fe/src/pages/layouts/WebsiteLayout.js:114-117`

```jsx
                <a href="mailto:vyvnt@fpt.edu.vn">
                  <FontAwesomeIcon icon={faEnvelope} />
                  <span className="pl-1">Contact</span>
                </a>
```

> 📖 **Thuật ngữ:** *`NavLink`* là anh em của `Link`, hoạt động y hệt nhưng **tự gắn class `active`** khi URL khớp `to`. Dự án dùng `NavLink` cho menu chính (`WebsiteLayout.js:239`), `Link` cho phần còn lại.

---

## 2. Soi code thật: `createBrowserRouter` + `RouterProvider`

`yotea-fe/src/App.js:1-2`

```js
import React from "react";
import { RouterProvider, createBrowserRouter } from "react-router-dom";
```

Đây là **Data Router API** của React Router v6.4+. Nếu tra Google thấy `<BrowserRouter><Routes><Route/></Routes></BrowserRouter>` thì đó là cách **cũ**. Yotea dùng `react-router-dom@^6.22.3` (`yotea-fe/package.json:21`) nên đi theo cách mới.

`yotea-fe/src/App.js:58-67`

```jsx
const App = () => {
  const router = createBrowserRouter([
    {
      path: "",
      element: <WebsiteLayout />,
      children: [
        {
          path: "",
          element: <HomePage />,
        },
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 58 | `const App = () => {` | `App` là một component bình thường |
| 59 | `createBrowserRouter([` | Nhận vào **mảng object** mô tả cây route; mỗi object là một nhánh |
| 61 | `path: ""` | Đường dẫn **rỗng** — gốc website, tức `/` |
| 62 | `element: <WebsiteLayout />` | Component vẽ ra khi khớp nhánh này |
| 63 | `children: [` | Danh sách route **con** nằm bên trong layout |
| 65 | `path: ""` (lần 2) | Route con rỗng = **index route**, khớp khi URL đúng bằng `/` |
| 66 | `element: <HomePage />` | Trang chủ được nhét vào `<Outlet />` của `WebsiteLayout` |

`yotea-fe/src/App.js:350-361`

```jsx
  ]);

  return (
    <>
      <RouterProvider router={router} />

      <ToastContainer />
    </>
  );
};

export default App;
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 350 | `]);` | Đóng mảng route và lời gọi `createBrowserRouter` |
| 353 | `<>` | Fragment — bọc 2 phần tử mà không đẻ thêm `div` thừa |
| 354 | `<RouterProvider router={router} />` | **Chỗ router thật sự chạy** — lắng nghe URL và vẽ nhánh khớp |
| 356 | `<ToastContainer />` | Chỗ `react-toastify` bắn thông báo; đặt **ngoài** router nên mọi trang dùng được |
| 361 | `export default App;` | Xuất cho `index.js` |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `createBrowserRouter` được gọi **bên trong** thân hàm `App` (dòng 59) ⇒ mỗi lần `App` re-render, cây 57 route bị tạo lại từ đầu. Cách chuẩn là đưa nó ra **ngoài** component, ở phạm vi module (đoạn dưới bạn tự viết thêm, dự án chưa có):
>
> ```js
> const router = createBrowserRouter([ /* ... */ ]);
> const App = () => (
>   <>
>     <RouterProvider router={router} />
>     <ToastContainer />
>   </>
> );
> ```

---

## 3. `path: ""`, `children` và `<Outlet />`

### 3.1. `path: ""` có hai nghĩa, tuỳ vị trí

| Vị trí | Nghĩa | Ví dụ |
|---|---|---|
| Ở **nhánh gốc** (`App.js:61`) | "Tôi không thêm gì vào URL, tôi chỉ là cái khung" | `WebsiteLayout` áp cho mọi URL không bắt đầu bằng `/admin` |
| Ở **route con** (`App.js:65`) | **Index route** — khớp khi URL dừng đúng ở đây | `/` → `HomePage`; `/my-account` → `UpdateInfoPage`; `/admin` → `Dashboard` |

Quy tắc vàng: **URL cuối cùng = path của cha nối với path của con.**

```
cha ""       + con "thuc-don"   =  /thuc-don
cha "/admin" + con "product" + cháu ":id/edit"  =  /admin/product/663f…/edit
```

### 3.2. `<Outlet />` — cái "lỗ" để nhét route con vào

Layout cha không biết trước con nó là ai, nó chỉ chừa sẵn một chỗ trống tên `<Outlet />`.

`yotea-fe/src/pages/layouts/WebsiteLayout.js:380-382`

```jsx
      <main>
        <Outlet />
      </main>
```

Nằm **giữa** `</header>` (dòng 378) và `<footer` (dòng 384) → mọi trang website tự động có header + footer mà không phải import lại.

`yotea-fe/src/pages/layouts/MyAccountLayout.js:72-74`

```jsx
        <div className="col-span-12 lg:col-span-9">
          <Outlet />
        </div>
```

Nằm ở **cột phải** của lưới 12 cột; cột trái (`aside`, dòng 25-71) là menu "Thông tin tài khoản / Đổi mật khẩu / Đơn hàng / Đăng xuất".

`yotea-fe/src/pages/layouts/AdminLayout.js:225-227`

```jsx
          <main>
            <Outlet />
          </main>
```

Nằm dưới header admin, bên phải sidebar quản trị.

### 3.3. Lồng ba tầng — trường hợp khó nhất

Khi vào `/my-account/update-password`:

```
<WebsiteLayout>                          ← App.js:62
   header
   <main><Outlet> ──► <PrivateRouter page="user">      ← App.js:158-162
                         <MyAccountLayout>             ← App.js:160
                            aside (menu tài khoản)
                            <Outlet> ──► <UpdatePasswordPage />   ← App.js:170
                         </MyAccountLayout>
                      </PrivateRouter>
   </main>
   footer
</WebsiteLayout>
```

`yotea-fe/src/App.js:156-163`

```jsx
        {
          path: "my-account",
          element: (
            <PrivateRouter page="user">
              <MyAccountLayout />
            </PrivateRouter>
          ),
          children: [
```

Vì `PrivateRouter` bọc **cả layout**, nên **mọi route con** của `my-account` đều được bảo vệ — không phải bọc lại từng trang. Đó là lợi ích lớn nhất của layout lồng nhau ([Bài 24](24-private-router.md)).

Nhánh admin làm y hệt — `yotea-fe/src/App.js:188-195`:

```jsx
    {
      path: "/admin",
      element: (
        <PrivateRouter page="admin">
          <AdminLayout />
        </PrivateRouter>
      ),
      children: [
```

> 💡 Nhánh admin là **nhánh gốc thứ hai**, ngang hàng với nhánh `""` chứ không nằm trong nó — vì thế trang admin **không có** header/footer website.

---

## 4. Bảng route đầy đủ của Yotea

### 4.1. Nhóm A — Website công khai (trong `<WebsiteLayout />`)

| Path khai báo | URL thực tế | Component | Layout cha | PrivateRouter? | Dòng |
|---|---|---|---|---|---|
| `""` | `/` | `HomePage` | WebsiteLayout | Không | `App.js:64-67` |
| `gioi-thieu` | `/gioi-thieu` | `AboutPage` | WebsiteLayout | Không | `:68-71` |
| `thuc-don` | `/thuc-don` | `ProductPage` | WebsiteLayout | Không | `:72-75` |
| `thuc-don/page/:page` | `/thuc-don/page/2` | `ProductPage` | WebsiteLayout | Không | `:76-79` |
| `danh-muc/:slug` | `/danh-muc/tra-sua` | `ProductByCate` | WebsiteLayout | Không | `:80-83` |
| `danh-muc/:slug/page/:page` | `/danh-muc/tra-sua/page/2` | `ProductByCate` | WebsiteLayout | Không | `:84-87` |
| `tim-kiem/:keyword` | `/tim-kiem/matcha` | `ProductSearchPage` | WebsiteLayout | Không | `:88-91` |
| `tim-kiem/:keyword/page/:page` | `/tim-kiem/matcha/page/2` | `ProductSearchPage` | WebsiteLayout | Không | `:92-95` |
| `san-pham/:slug` | `/san-pham/tra-sua-tran-chau` | `ProductDetailPage` | WebsiteLayout | Không | `:96-99` |
| `san-pham/:slug/page/:page` | `/san-pham/abc/page/2` | `ProductDetailPage` | WebsiteLayout | Không | `:100-103` |
| `tin-tuc` | `/tin-tuc` | `NewsPage` | WebsiteLayout | Không | `:104-107` |
| `tin-tuc/page/:page` | `/tin-tuc/page/2` | `NewsPage` | WebsiteLayout | Không | `:108-111` |
| `tin-tuc/:slug` | `/tin-tuc/khuyen-mai` | `NewsByCatePage` | WebsiteLayout | Không | `:112-115` |
| `tin-tuc/:slug/page/:page` | `/tin-tuc/khuyen-mai/page/2` | `NewsByCatePage` | WebsiteLayout | Không | `:116-119` |
| `bai-viet/:slug` | `/bai-viet/tin-a` | `NewsDetail` | WebsiteLayout | Không | `:120-123` |
| `lien-he` | `/lien-he` | `ContactPage` | WebsiteLayout | Không | `:124-127` |
| `cua-hang` | `/cua-hang` | `StorePage` | WebsiteLayout | Không | `:128-131` |
| `login` | `/login` | `LoginPage` | WebsiteLayout | Không | `:132-135` |
| `register` | `/register` | `RegisterPage` | WebsiteLayout | Không | `:136-139` |
| `forgot` | `/forgot` | `ForgotPage` | WebsiteLayout | Không | `:140-143` |
| `cart` | `/cart` | `CartPage` | WebsiteLayout | Không | `:144-147` |
| `checkout` | `/checkout` | `CheckoutPage` | WebsiteLayout | **Không** ⚠️ | `:148-151` |
| `thank-you` | `/thank-you` | `ThankPage` | WebsiteLayout | Không | `:152-155` |

### 4.2. Nhóm B — Tài khoản của tôi (lồng 2 tầng)

| Path khai báo | URL thực tế | Component | Layout cha | PrivateRouter? | Dòng |
|---|---|---|---|---|---|
| `""` | `/my-account` | `UpdateInfoPage` | WebsiteLayout → MyAccountLayout | **Có** (`page="user"`) | `App.js:164-167` |
| `update-password` | `/my-account/update-password` | `UpdatePasswordPage` | như trên | Kế thừa | `:168-171` |
| `cart` | `/my-account/cart` | `MyCartPage` | như trên | Kế thừa | `:172-175` |
| `cart/page/:page` | `/my-account/cart/page/2` | `MyCartPage` | như trên | Kế thừa | `:176-179` |
| `cart/:id` | `/my-account/cart/663f…` | `MyCartDetailPage` | như trên | Kế thừa | `:180-183` |

### 4.3. Nhóm C — Quản trị (trong `<AdminLayout />`)

Cả nhánh bọc `<PrivateRouter page="admin">` ở gốc (`App.js:190-194`) ⇒ **mọi dòng đều CÓ PrivateRouter**, layout cha đều là **AdminLayout**.

| Nhóm | Path con | URL thực tế | Component | Dòng |
|---|---|---|---|---|
| (gốc) | `""` | `/admin` | `Dashboard` | `App.js:196-199` |
| `user` | `""` · `page/:page` · `add` · `:id/edit` | `/admin/user…` | `UserListPage` ×2 · `AddUserPage` · `EditUserPage` | `:203-218` |
| `news` | `""` · `page/:page` · `add` · `:id/edit` | `/admin/news…` | `NewsListPage` ×2 · `AddNewsPage` · `EditNewsPage` | `:224-239` |
| `category-news` | `""` · `add` · `:id/edit` | `/admin/category-news…` | `CateNewsListPage` · `AddCateNewsPage` · `EditCateNewsPage` | `:245-256` |
| `product` | `""` · `page/:page` · `add` · `:id/edit` | `/admin/product…` | `ProductListPage` ×2 · `AddProductPage` · `EditProductPage` | `:262-277` |
| `slider` | `""` · `add` · `:id/edit` | `/admin/slider…` | `SliderListPage` · `AddSlidePage` · `EditSlidePage` | `:283-294` |
| `category` | `""` · `add` · `:id/edit` | `/admin/category…` | `CategoryListPage` · `AddCategoryPage` · `EditCategoryPage` | `:300-311` |
| `cart` | `""` · `page/:page` · `:id/detail` | `/admin/cart…` | `CartListPage` ×2 · `CartDetailPage` | `:317-328` |
| `contact` | `""` · `page/:page` · `:id/detail` | `/admin/contact…` | `ContactListPage` ×2 · `ContactDetailPage` | `:334-345` |

**Tổng: 23 route website + 5 route my-account + 29 route admin.**

> 💡 Các node như `{ path: "user", children: [...] }` (`App.js:200-220`) **không có `element`** — chúng chỉ là **cái hộp gom nhóm URL** (*pathless grouping*), giúp khỏi gõ lặp `user/` ở mọi route con.

---

## 5. Route động và `useParams()`

| Ký hiệu | Ý nghĩa trong Yotea | Ví dụ URL | Đọc bằng |
|---|---|---|---|
| `:slug` | Đường dẫn thân thiện SEO của sản phẩm / danh mục / bài viết | `/san-pham/tra-sua-tran-chau` | `useParams().slug` |
| `:keyword` | Từ khoá gõ vào ô tìm kiếm | `/tim-kiem/matcha` | `useParams().keyword` |
| `:page` | Số trang khi phân trang | `/thuc-don/page/3` | `useParams().page` |
| `:id` | `_id` MongoDB (24 ký tự hex) | `/my-account/cart/663f1a…` | `useParams().id` |

> 💡 Trang **cho người dùng** cần URL đẹp nên dùng `:slug` (bạn đã sinh slug ở [Bài 08](08-slug-slugify.md)); trang **quản trị** không cần SEO nên dùng thẳng `:id`.

`yotea-fe/src/pages/user/NewsDetail.js:14-26`

```jsx
const NewsDetail = () => {
  const [news, setNews] = useState();

  const { slug } = useParams();

  useEffect(() => {
    const getNews = async () => {
      const { data } = await get(slug);
      setNews(data);
      updateTitle(`${data.title}`);
    };
    getNews();
  }, [slug]);
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 17 | `const { slug } = useParams()` | `useParams()` trả object `{ slug: "tin-a" }`, destructuring lấy ngay ra |
| 21 | `await get(slug)` | Gọi `GET /api/news/tin-a` |
| 26 | `}, [slug]);` | **Có `slug` trong dependency array** — bấm sang bài khác thì effect chạy lại, tải bài mới |

> ⚠️ Quên `[slug]` mà để `[]` là dính bug kinh điển: URL đổi nhưng nội dung **đứng yên**. `ProductSearchPage.js:11-13` đang mắc đúng lỗi này với `updateTitle` — tìm từ khoá mới mà tiêu đề tab không đổi.

Trang tìm kiếm lấy **hai** tham số — `yotea-fe/src/pages/user/ProductSearchPage.js:8-9`:

```jsx
const ProductSearchPage = () => {
  const { keyword, page } = useParams();
```

Trang chi tiết đơn hàng lấy `:id` — `yotea-fe/src/pages/user/my-account/MyCartDetailPage.js:8-12`:

```jsx
const MyCartDetailPage = () => {
  const [order, setOrder] = useState({});
  const [orderDetail, setOrderDetail] = useState();

  const { id } = useParams();
```

### 5.1. Kiểu "route lặp đôi" mà Yotea dùng cho phân trang

Nhìn bảng route sẽ thấy một khuôn mẫu lặp lại: `thuc-don` (`App.js:72-75`) và `thuc-don/page/:page` (`App.js:76-79`) — **cùng một component, khai hai lần**. Vì `:page` là tham số **bắt buộc**: nếu chỉ khai `thuc-don/page/:page` thì URL `/thuc-don` (không có số trang) **không khớp route nào**.

Component xử lý việc thiếu `page` như sau — `yotea-fe/src/pages/user/ProductPage.js:8-9`:

```jsx
const ProductPage = () => {
  const { page } = useParams();
```

`yotea-fe/src/pages/user/ProductPage.js:40-44`

```jsx
        <ProductContent
          getProducts={getAll}
          page={Number(page) || 1}
          url="thuc-don"
        />
```

| URL | `page` | `Number(page)` | `Number(page) \|\| 1` |
|---|---|---|---|
| `/thuc-don` | `undefined` | `NaN` | **`1`** |
| `/thuc-don/page/3` | `"3"` (chuỗi!) | `3` | `3` |
| `/thuc-don/page/abc` | `"abc"` | `NaN` | **`1`** |

> 💡 `useParams()` **luôn trả về chuỗi**, kể cả khi trông như số. Quên `Number()` là dính `"3" + 1 === "31"`.

Link phân trang được sinh ở `yotea-fe/src/components/user/Pagination.js:5-11`:

```jsx
const Pagination = ({ page, totalPage, url }) => {
  const pagination = [];
  for (let i = 1; i <= totalPage; i++) {
    pagination.push(
      <li key={i}>
        <Link
          to={`/${url}/page/${i}`}
```

Với `url="thuc-don"`, vòng lặp đẻ ra `/thuc-don/page/1`, `/thuc-don/page/2`…

### 5.2. Path param vs query string

Cách kia (dự án **không** dùng) là `/thuc-don?page=2`, đọc bằng hook `useSearchParams()`.

| | Path param `/thuc-don/page/2` (Yotea dùng) | Query string `/thuc-don?page=2` |
|---|---|---|
| URL đẹp, dễ đọc | ✅ Rất đẹp | ❌ Có `?` và `=` |
| SEO | ✅ Google index như URL riêng biệt | ⚠️ Dễ bị coi là nội dung trùng lặp |
| Số route phải khai | ❌ **Gấp đôi** | ✅ Chỉ **một** |
| Thêm bộ lọc thứ 2 (giá, sắp xếp) | ❌ Bế tắc: `/thuc-don/page/2/sort/price/order/asc`? | ✅ `?page=2&sort=price&order=asc` |
| Phụ thuộc thứ tự tham số | ❌ Có | ✅ Không |

**Kết luận:** Yotea chọn path param vì URL đẹp và SEO tốt — hợp lý cho trang bán hàng. Cái giá là **8 trong 23 route website chỉ là bản sao khác mỗi `/page/:page`**. Khi thêm bộ lọc giá + sắp xếp, path param sẽ nổ tung và buộc phải chuyển sang query string.

---

## 6. Ba hook điều hướng

**`useParams()`** — đã xem ở mục 5, có mặt ở 22 file của dự án.

**`useNavigate()`** — chuyển trang bằng code, dùng khi việc chuyển xảy ra **sau một hành động** chứ không do người dùng bấm link.

`yotea-fe/src/pages/layouts/WebsiteLayout.js:85-93`

```jsx
  const navigate = useNavigate();
  const handleSubmitFormSearch = (e) => {
    e.preventDefault();
    if (!keyword) {
      toast.error("Vui lòng nhập tên SP");
    } else {
      navigate(`/tim-kiem/${keyword}`);
    }
  };
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 85 | `const navigate = useNavigate()` | Lấy **hàm** điều hướng; phải gọi ở thân component, không được gọi trong `if` |
| 87 | `e.preventDefault()` | Chặn hành vi mặc định của form (nạp lại trang) — **bắt buộc**, quên là SPA hỏng |
| 88-89 | `if (!keyword)` | Chưa gõ gì thì báo lỗi, **không** chuyển trang |
| 91 | `navigate(\`/tim-kiem/${keyword}\`)` | Chuyển trang y hệt bấm `<Link>` |

Sau khi đăng nhập — `yotea-fe/src/pages/auth/LoginPage.js:45-51`:

```jsx
        toast.success("Đăng nhập thành công");

        if (data.user.role) {
          navigate("/admin");
        } else {
          navigate("/");
        }
```

Việc này **không thể** làm bằng `<Link>` vì lúc viết JSX ta chưa biết người đăng nhập là ai. `useNavigate` còn dùng để chặn vào trang không hợp lệ — `yotea-fe/src/pages/auth/LoginPage.js:58-62`:

```jsx
  useEffect(() => {
    updateTitle("Đăng nhập");

    if (isLogged) navigate("/");
  }, []);
```

Vài dạng gọi khác nên nhớ (dự án chưa dùng): `navigate(-1)` quay lại như nút Back · `navigate("/cart", { replace: true })` thay thế lịch sử · `navigate("/thank-you", { state: { orderId } })` gửi kèm dữ liệu.

**`useLocation()`** — cho biết đang ở đâu: `location.pathname` (`/thuc-don/page/2`), `location.search` (`?sort=price`), `location.hash`, `location.state`.

> ⚠️ **Yotea KHÔNG dùng `useLocation` ở bất kỳ file nào** — bỏ lỡ hai cải tiến đáng làm:
> 1. **Cuộn lên đầu khi đổi route.** Hiện bấm phân trang xong trang vẫn nằm giữa chừng. Sửa (bạn tự viết thêm): `const { pathname } = useLocation();` rồi `useEffect(() => window.scrollTo(0, 0), [pathname]);`
> 2. **Quay lại đúng trang cũ sau khi đăng nhập.** `PrivateRouter` hiện đá thẳng về `/login` mà không nhớ người dùng muốn vào đâu. Chuẩn hơn: `<Navigate to="/login" state={{ from: location }} />` rồi `navigate(location.state?.from ?? "/")` — bàn kỹ ở [Bài 24](24-private-router.md).

---

## 7. Route tiếng Việt — không chỉ cho đẹp

| Route | Nghĩa | Component |
|---|---|---|
| `/thuc-don` | Thực đơn | `ProductPage` |
| `/danh-muc/:slug` | Danh mục sản phẩm | `ProductByCate` |
| `/san-pham/:slug` | Sản phẩm (chi tiết) | `ProductDetailPage` |
| `/tim-kiem/:keyword` | Tìm kiếm | `ProductSearchPage` |
| `/tin-tuc` | Tin tức | `NewsPage` |
| `/bai-viet/:slug` | Bài viết (chi tiết) | `NewsDetail` |
| `/gioi-thieu` | Giới thiệu | `AboutPage` |
| `/lien-he` | Liên hệ | `ContactPage` |
| `/cua-hang` | Cửa hàng | `StorePage` |

**Bốn lợi ích thật sự:**

1. **Từ khoá nằm ngay trong URL.** Google chấm điểm cả đường dẫn — người Việt gõ "thực đơn trà sữa" thì URL chứa `thuc-don` khớp tốt hơn hẳn `/menu`.
2. **Người dùng đọc URL là hiểu.** Thấy `yotea.vn/tin-tuc/khuyen-mai` là biết ngay ở mục nào → tăng tỉ lệ bấm khi link được chia sẻ lên Facebook/Zalo.
3. **Không dấu, chữ thường, gạch nối** — chính là lý do backend phải sinh slug bằng `slugify` ([Bài 08](08-slug-slugify.md)). URL có dấu bị mã hoá thành `%E1%BA%A1%C3%A0…`, xấu và dễ vỡ khi copy-paste.
4. **Gạch nối `-` chứ không phải gạch dưới `_`.** Google coi `-` là dấu cách giữa các từ; `_` thì dính liền — `tra-sua` được hiểu là "tra sua", còn `tra_sua` bị hiểu là một từ vô nghĩa.

> ⚠️ **Nhưng dự án làm chưa nhất quán:** vẫn còn `/login`, `/register`, `/forgot`, `/cart`, `/checkout`, `/thank-you`, `/my-account` bằng tiếng Anh. Chính sự lẫn lộn này đẻ ra một bug thật — xem ngay mục 8.

---

## 8. ⚠️ Ba lỗi routing có thật trong dự án

### 8.1. Không có `errorElement`, không có route `"*"`

Cả 362 dòng `App.js` **không hề xuất hiện chữ `errorElement`**, cũng không có `{ path: "*" }`. Hệ quả: gõ `localhost:3000/khong-ton-tai` ra **trang lỗi mặc định xấu xí của React Router** ("Unexpected Application Error!") — trắng trơn, không header, không footer, không nút quay về, lại lộ chi tiết kỹ thuật.

Cách sửa (bạn tự viết thêm, dự án chưa có) — thêm route bắt-tất-cả vào cuối `children` của `WebsiteLayout`, hoặc gắn `errorElement` ở nhánh gốc:

```jsx
{ path: "*", element: <NotFoundPage /> },          // bắt URL không khớp
// hoặc, ở nhánh gốc:
{ path: "", element: <WebsiteLayout />, errorElement: <ErrorPage />, children: [ /* ... */ ] },
```

Khác biệt: `path: "*"` chỉ bắt **URL không khớp**; `errorElement` bắt **cả lỗi runtime** văng ra từ bất kỳ route con nào — an toàn hơn nhiều.

### 8.2. Hai link chết

`yotea-fe/src/pages/layouts/WebsiteLayout.js:298`

```jsx
                  <Link to="/gio-hang" className="relative">
```

Route thật là `cart` (`App.js:144-147`), **không có** route nào tên `gio-hang`. Đây là icon giỏ hàng **bản mobile** (nằm trong `<ul className="flex flex-1 justify-end md:hidden">`, dòng 285) ⇒ thu nhỏ cửa sổ dưới 768px, bấm giỏ hàng → **trang trắng**. Bản desktop ở dòng 216 lại trỏ đúng `/cart` nên bug lọt lưới suốt vì chỉ test trên màn hình lớn.

`yotea-fe/src/pages/layouts/AdminLayout.js:208-213`

```jsx
                  <Link
                    to="/admin/profile"
                    className="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100"
                  >
                    Profile
                  </Link>
```

Rà toàn bộ nhánh admin (`App.js:188-349`) **không có route `profile`**. Buồn cười hơn: thư mục `yotea-fe/src/pages/admin/profile/` **có tồn tại** với đủ `AdminUpdateInfoPage.js` và `AdminUpdatePassword.js` — code viết xong nhưng **quên khai route**.

### 8.3. `/checkout` không được bọc `PrivateRouter`

`yotea-fe/src/App.js:148-151`

```jsx
        {
          path: "checkout",
          element: <CheckoutPage />,
        },
```

Trang **thanh toán** nằm chung nhóm với các trang công khai. Ai cũng gõ thẳng `localhost:3000/checkout` vào được; `/cart` và `/thank-you` cũng vậy.

> 🔒 **Ghi chú bảo mật:** đây chưa phải lỗ hổng chết người vì backend vẫn kiểm tra token khi tạo đơn. Nhưng nó là **trải nghiệm tồi** (khách vào tới trang thanh toán rồi mới bị chặn) và là dấu hiệu dự án không có chiến lược phân quyền nhất quán ở tầng route — mổ xẻ ở [Bài 24](24-private-router.md) và [Bài 33](33-ra-soat-bao-mat.md). Lưu ý: Yotea **cố ý** cho khách vãng lai đặt hàng (`userId: (user && user._id) || ""`), nên `/checkout` mở tự do có thể là chủ đích — nhưng phải ghi rõ trong tài liệu chứ không để người đọc tự đoán.

---

## 9. 🛠️ Tự tay làm — thêm route `/topping`

> **Mục tiêu:** cuối phần, gõ `localhost:3000/topping` sẽ ra trang **có đủ header + footer Yotea**, ở giữa hiện một dòng chữ; trên menu cũng có thêm mục "Topping" bấm được.

Ở phần backend bạn đã dựng xong API Topping (model, route, controller, CRUD, slug, bộ lọc query, phân quyền, Swagger). **Bài này ta làm tiếp bước đầu tiên của phía giao diện: mở một cánh cửa URL cho chức năng đó.** Phòng còn trống cũng không sao — ta nhồi dần ở bài 16 → 22.

### Bước 1 — Tạo trang tạm

```jsx
// yotea-fe/src/pages/user/ToppingPage.js  ← file MỚI, bạn tự tạo
import { useEffect } from "react";
import { updateTitle } from "../../utils";

const ToppingPage = () => {
  useEffect(() => {
    updateTitle("Topping");
  }, []);

  return (
    <section className="container max-w-6xl mx-auto px-3 py-16 text-center">
      <h1 className="text-3xl font-semibold text-[#D9A953]">
        Trang Topping đang được xây dựng
      </h1>
      <p className="mt-2 text-gray-500">
        Route /topping đã hoạt động — bài 16 sẽ vẽ danh sách topping vào đây.
      </p>
    </section>
  );
};

export default ToppingPage;
```

Vài chi tiết cố ý bắt chước dự án: `updateTitle` đổi tiêu đề tab (`utils/index.js:42-44`); bộ class khung `container max-w-6xl mx-auto px-3` giống `ProductPage.js:17`; `text-[#D9A953]` là màu vàng đồng thương hiệu; `export default` để `App.js` import không cần ngoặc nhọn.

### Bước 2 — Import vào `App.js`

Trong khối import (dòng 1–56) của `yotea-fe/src/App.js`, thêm **một dòng** (đặt sau dòng 22 cho gọn):

```js
// yotea-fe/src/App.js — dòng bạn tự thêm
import ToppingPage from "./pages/user/ToppingPage";
```

### Bước 3 — Khai route vào `children` của `WebsiteLayout`

Tìm route `thuc-don` (dòng 72-75), thêm khối sau **ngay dưới** nó — tức là **bên trong mảng `children` của nhánh `path: ""`**:

```jsx
// yotea-fe/src/App.js — khối bạn tự thêm, đặt trong children của nhánh path: ""
        {
          path: "topping",
          element: <ToppingPage />,
        },
```

⚠️ **Ba kiểu chèn sai chỗ rất hay gặp:**

| Chèn nhầm vào đâu | Hậu quả |
|---|---|
| Ngoài mảng gốc, ngang hàng `{ path: "" }` và `{ path: "/admin" }` | Trang hiện ra nhưng **mất header và footer** |
| Trong `children` của `my-account` (dòng 163-184) | URL thành `/my-account/topping` và **bắt phải đăng nhập** |
| Trong `children` của `/admin` (dòng 195-348) | URL thành `/admin/topping`, layout admin, chỉ admin xem được |

Viết `path: "topping"` **không có `/` ở đầu** — vì nó nối vào `path: ""` của cha. Viết `"/topping"` cũng chạy ở đây (do cha rỗng) nhưng là thói quen xấu: với cha `/admin` thì `path: "/product"` sẽ **phá vỡ** việc nối chuỗi.

### Bước 4 — Thêm `<Link to="/topping">` vào menu

Menu bên trái trong `yotea-fe/src/pages/layouts/WebsiteLayout.js` nằm ở dòng 237-264, mỗi mục có dạng như `WebsiteLayout.js:238-240`:

```jsx
                <li className="menu__item pr-4 font-semibold text-gray-500 transition ease-linear duration-200 hover:text-black">
                  <NavLink to="/">Trang chủ</NavLink>
                </li>
```

Thêm mục mới **ngay sau** mục "Giới thiệu" (kết thúc ở dòng 243):

```jsx
{/* yotea-fe/src/pages/layouts/WebsiteLayout.js — khối bạn tự thêm */}
                <li className="menu__item pr-4 font-semibold text-gray-500 transition ease-linear duration-200 hover:text-black">
                  <Link to="/topping">Topping</Link>
                </li>
```

> 💡 Cả `Link` lẫn `NavLink` **đã được import sẵn** ở `WebsiteLayout.js:23` nên không phải sửa dòng import. Muốn mục menu tự sáng lên khi đang ở trang Topping thì đổi thành `<NavLink to="/topping">`.

---

## 10. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-fe
npm start
```

CRA tự nạp lại (hot reload), không cần tắt bật lại. **Phải đúng cả 5 điểm:**

| # | Việc làm | Kết quả bắt buộc thấy |
|---|---|---|
| 1 | Gõ thẳng `http://localhost:3000/topping` | Có **header** (thanh vàng + logo + menu), giữa là dòng *"Trang Topping đang được xây dựng"*, dưới là **footer** đầy đủ |
| 2 | Nhìn tiêu đề tab | `Topping - Trà sữa Yotea` |
| 3 | Từ trang chủ bấm mục **Topping** | Chuyển trang **tức thì**, không nháy trắng, không có vòng xoay loading của trình duyệt |
| 4 | Thêm 1 sản phẩm vào giỏ → bấm Topping → bấm Trang chủ | Số trên icon giỏ hàng **giữ nguyên** ⇒ Redux store không bị reset |
| 5 | Mở DevTools tab **Network**, xoá log rồi bấm menu Topping | **Không có request nào** kiểu `topping` / `index.html` ⇒ đúng là điều hướng SPA |

**Kiểm tra ngược (rất đáng làm một lần):** tạm đổi link vừa thêm thành `<a href="/topping">Topping</a>` rồi lặp lại điểm 4 và 5. Bạn sẽ thấy Network tải lại cả bundle, màn hình chớp trắng ~1 giây, số giỏ hàng **vẫn còn** (nhờ redux-persist) nhưng danh sách yêu thích ❤️ đã **về 0** — vì `wishlist` không nằm trong whitelist persist. Xong nhớ **đổi lại `<Link>`**.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Module not found: Can't resolve './pages/user/ToppingPage'` | Sai tên/đường dẫn import | Kiểm tra **chữ hoa/thường** (`ToppingPage` ≠ `toppingpage`). Windows bỏ qua nhưng build trên Linux thì lỗi |
| `useNavigate() may be used only in the context of a <Router>` | Gọi hook ở component nằm **ngoài** `RouterProvider` | Đưa component vào trong cây route |
| Trang trắng, console báo `Unexpected Application Error!` | URL không khớp route nào, dự án lại không có `errorElement` | Soi lại `path` trong `App.js` (nhớ bug `/gio-hang` mục 8.2) |
| Trang hiện ra nhưng **mất header/footer** | Route bị chèn **ngoài** `children` của `WebsiteLayout` | Chuyển khối route vào đúng `children` của nhánh `path: ""` |
| Đổi URL nhưng nội dung **không đổi** | `useEffect` thiếu tham số động trong dependency array | Sửa `}, []);` thành `}, [slug]);` |
| `page` bị cộng thành `"31"` thay vì `4` | `useParams()` trả về **chuỗi** | Bọc `Number(page)` như `ProductPage.js:42` |
| Submit form xong trang tự tải lại | Quên `e.preventDefault()` | Thêm vào đầu handler (xem `WebsiteLayout.js:87`) |
| Deploy xong, gõ thẳng `/topping` ra **404 của server** | Server chưa rewrite mọi URL về `index.html` | Cấu hình `try_files` (Nginx) / `_redirects` (Netlify) — [Bài 36](36-build-va-deploy.md) |

---

## 12. 📝 Bài tập

**Bài 1.** Không mở trình duyệt, chỉ nhìn `App.js`, cho biết mỗi URL sau vẽ ra component nào (hoặc lỗi gì): (1) `/danh-muc/tra-sua/page/3` · (2) `/tin-tuc/khuyen-mai` · (3) `/bai-viet/khuyen-mai` · (4) `/my-account/cart/663f1a2b3c4d5e6f7a8b9c0d` · (5) `/admin/product/add` · (6) `/topping/page/2` (sau khi đã làm phần Tự tay làm).

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

1. `WebsiteLayout` → `ProductByCate` (`App.js:84-87`); `useParams()` = `{ slug: "tra-sua", page: "3" }`.
2. `WebsiteLayout` → `NewsByCatePage` (`App.js:112-115`) — danh sách bài viết **thuộc chuyên mục** `khuyen-mai`.
3. `WebsiteLayout` → `NewsDetail` (`App.js:120-123`) — **một bài viết** có slug `khuyen-mai`. Khác biệt tinh tế giữa câu 2 và 3: `tin-tuc/:slug` là *chuyên mục*, `bai-viet/:slug` là *bài viết*. Đặt tên route rõ như vậy là điểm cộng của dự án.
4. `WebsiteLayout` → `PrivateRouter(user)` → `MyAccountLayout` → `MyCartDetailPage` (`App.js:180-183`) — lồng **ba** tầng. Chưa đăng nhập thì bị đá về `/login`.
5. `PrivateRouter(admin)` → `AdminLayout` → `AddProductPage` (`App.js:270-273`). **Không** có `WebsiteLayout`.
6. **Không khớp route nào** → trang lỗi mặc định. Bạn mới khai `path: "topping"`, chưa khai `topping/page/:page` — đúng hệ quả của kiểu route lặp đôi ở mục 5.1.

</details>

**Bài 2.** Khai thêm route để `/topping/page/2` chạy được, và sửa `ToppingPage` in ra số trang hiện tại; không có số trang thì mặc định là 1.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thêm khối route thứ hai ngay dưới khối `topping` trong `App.js` (bạn tự viết thêm):

```jsx
        {
          path: "topping/page/:page",
          element: <ToppingPage />,
        },
```

Rồi sửa `yotea-fe/src/pages/user/ToppingPage.js` (file của bạn):

```jsx
import { useEffect } from "react";
import { useParams } from "react-router-dom";
import { updateTitle } from "../../utils";

const ToppingPage = () => {
  const { page } = useParams();
  const currentPage = Number(page) || 1;

  useEffect(() => {
    updateTitle(`Topping - Trang ${currentPage}`);
  }, [currentPage]);

  return (
    <section className="container max-w-6xl mx-auto px-3 py-16 text-center">
      <h1 className="text-3xl font-semibold text-[#D9A953]">
        Topping — trang {currentPage}
      </h1>
    </section>
  );
};

export default ToppingPage;
```

Hai điểm đáng nhớ: `Number(page) || 1` bắt chước y hệt `ProductPage.js:42`, xử lý gọn cả `undefined` lẫn `abc`; và `}, [currentPage]);` khiến tiêu đề tab đổi theo trang — đúng chỗ mà `ProductSearchPage.js:11-13` làm thiếu.

</details>

**Bài 3.** Dự án thiếu trang 404 tử tế. Hãy tự viết `NotFoundPage` và gắn vào router sao cho: gõ URL sai vẫn **giữ nguyên header + footer**, hiện thông báo thân thiện và có nút quay về trang chủ.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Tạo file mới `yotea-fe/src/pages/user/NotFoundPage.js` (bạn tự viết, dự án chưa có):

```jsx
import { useEffect } from "react";
import { Link } from "react-router-dom";
import { updateTitle } from "../../utils";

const NotFoundPage = () => {
  useEffect(() => {
    updateTitle("Không tìm thấy trang");
  }, []);

  return (
    <section className="container max-w-6xl mx-auto px-3 py-20 text-center">
      <h1 className="text-6xl font-bold text-[#D9A953]">404</h1>
      <p className="mt-3 text-gray-500">
        Rất tiếc, trang bạn tìm không tồn tại hoặc đã bị xoá.
      </p>
      <Link to="/" className="inline-block mt-5 px-6 py-2 bg-[#D9A953] text-white rounded">
        Về trang chủ
      </Link>
    </section>
  );
};

export default NotFoundPage;
```

Trong `App.js`, import rồi thêm route **cuối cùng** trong `children` của nhánh `path: ""`:

```jsx
        {
          path: "*",
          element: <NotFoundPage />,
        },
```

**Vì sao đặt trong `children`?** Để `WebsiteLayout` vẫn bọc → còn header/footer, người dùng bấm menu đi tiếp được. Đặt ngoài cùng thì trang 404 trơ trọi, cụt đường.

**Vì sao đặt cuối mảng?** React Router v6 **chấm điểm độ cụ thể** chứ không xét theo thứ tự như v5, nên `"*"` luôn bị xếp cuối dù bạn đặt ở đâu — nhưng đặt cuối vẫn là thói quen tốt để người đọc hiểu ngay ý đồ.

**Nâng cao:** thêm `errorElement: <ErrorPage />` ở nhánh gốc để bắt luôn lỗi runtime (ví dụ `formatCurrency(undefined)` nổ trong một page), thứ mà `path: "*"` **không** bắt được.

</details>

**Bài 4.** Bug `/gio-hang` ở `WebsiteLayout.js:298` có **hai** cách sửa. Nêu cả hai, chỉ rõ phải sửa file/dòng nào, và cách nào tốt hơn.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Cách 1 — sửa link cho khớp route:** đổi `WebsiteLayout.js:298` từ `to="/gio-hang"` thành `to="/cart"`. Chỉ đụng **một dòng, một file**.

**Cách 2 — sửa route cho khớp phong cách tiếng Việt:** đổi `App.js:145` từ `path: "cart"` thành `path: "gio-hang"`, rồi rà toàn dự án sửa mọi chỗ trỏ `/cart` — ít nhất có `WebsiteLayout.js:216` — và **cẩn thận không nhầm** với `/my-account/cart` (`App.js:173`) hay `/admin/cart` (`App.js:315`), ba đường dẫn khác nhau cùng chứa chữ `cart`. Tìm nhanh bằng `grep -rn 'to="/cart"' src/` khi đứng ở `yotea-fe`.

**Cách nào tốt hơn?** Cần vá bug ngay: cách 1. Đang refactor có kế hoạch để URL đồng bộ tiếng Việt (đúng tinh thần mục 7): cách 2 — nhưng phải làm trọn gói cho cả `/login`, `/register`, `/checkout`, và thêm redirect từ URL cũ sang URL mới để không mất thứ hạng SEO. Bàn ở [Bài 34](34-refactor-du-an.md).

</details>

---

## 📌 Tóm tắt

- **SPA** = một file HTML duy nhất, JS vẽ đè nội dung. Đổi trang **không** nạp lại trình duyệt nên **Redux store được giữ nguyên**.
- `<Link to>` cho link **nội bộ**; `<a href>` chỉ cho link **ra ngoài**. Dùng nhầm là mất state và chậm.
- `createBrowserRouter([...])` nhận **mảng object**; `<RouterProvider router={router} />` mới là chỗ router thật sự chạy (`App.js:59` và `:354`).
- `path: ""` ở nhánh gốc = "chỉ là cái khung"; ở route con = **index route**. URL cuối cùng = **path cha nối path con**.
- `<Outlet />` là cái lỗ nhét route con vào: `WebsiteLayout.js:381`, `MyAccountLayout.js:73`, `AdminLayout.js:226`. Bọc `PrivateRouter` ở **layout cha** thì mọi route con tự động được bảo vệ.
- `useParams()` **luôn trả về chuỗi** — nhớ `Number()`. `useNavigate()` chuyển trang bằng code sau khi submit. `useLocation()` cho biết đang ở đâu (dự án **chưa** dùng — một thiếu sót).
- Route lặp đôi (`thuc-don` + `thuc-don/page/:page`) cho URL đẹp và SEO tốt nhưng phải khai gấp đôi, và sẽ bế tắc khi có nhiều bộ lọc — lúc đó phải chuyển sang query string.
- Route tiếng Việt không dấu, gạch nối `-` (không phải `_`) — tốt cho SEO và cho người đọc.
- **Ba lỗi có thật:** thiếu `errorElement` / route `"*"`; hai link chết `/gio-hang` (`WebsiteLayout.js:298`) và `/admin/profile` (`AdminLayout.js:209`); `/checkout` không bọc `PrivateRouter` (`App.js:148-151`).

**Từ khoá tra cứu thêm:** `react router v6 createBrowserRouter`, `nested routes Outlet`, `useParams useNavigate useLocation`, `index route react router`, `errorElement react router`, `splat route`, `SPA vs MPA`, `history pushState API`

➡️ **Bài tiếp theo:** [16 — Layout và component tái sử dụng](16-layout-va-component.md) — cánh cửa `/topping` đã mở nhưng phòng bên trong còn trống. Bài sau bạn sẽ tự viết `ToppingCard` và dựng `ToppingPage` thành một trang danh sách thật sự, học luôn cách chia component để khỏi copy-paste JSX.
