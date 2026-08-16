# Bài 16 — Layout và component tái sử dụng

> **Phần 2 · Frontend React** — Thời lượng ước tính: **~70 phút**
> ⬅️ Bài trước: [15 — React Router v6](15-routing-v6.md) · Bài sau: [17 — Tailwind CSS trong dự án](17-tailwind-css.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được **vì sao phải tách component**, và tách theo tiêu chí nào.
- Đọc hiểu **ba layout** của Yotea: `WebsiteLayout`, `AdminLayout`, `MyAccountLayout` — và biết `<Outlet />` nằm ở đâu trong mỗi cái.
- Truyền được dữ liệu xuống component bằng **props**, kể cả prop đặc biệt **`children`** và prop là **một hàm**.
- Phân biệt rành mạch **"page"** (gắn với route, tự lấy dữ liệu) và **"component"** (nhận props, chỉ hiển thị).
- Hiểu vì sao React cần **`key`** trong `.map()`, và tại sao `key={index}` là một quả bom hẹn giờ.
- Viết được **điều kiện hiển thị** trong JSX bằng `&&` / toán tử ba ngôi, và tránh được bẫy "số 0 hiện ra màn hình".
- Tự tay viết `ToppingCard.js` + `ToppingPage.js` và nhìn thấy danh sách topping trên trình duyệt.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 15 — React Router v6](15-routing-v6.md): bạn đã thêm route `topping` vào `App.js` trỏ tới trang `ToppingPage`.
- Đã đọc [Bài 03](03-kien-thuc-nen.md) phần destructuring và `map/filter/find` — bài này dùng lại liên tục.
- Frontend đang chạy: đứng tại `yotea-fe`, gõ `npm start`, mở `http://localhost:3000`.

> 💬 **Nối tiếp mạch thực hành:** ở bài trước bạn đã **khai báo route `topping`** trong `App.js` và tạo một trang `ToppingPage` rỗng chỉ để router không báo lỗi. Bài này ta làm tiếp: **viết component `ToppingCard` và đổ dữ liệu (tạm thời là dữ liệu giả) vào `ToppingPage`** để trang đó thực sự có hình hài.

---

## 1. Vì sao phải tách component?

Hãy tưởng tượng bạn viết toàn bộ website Yotea vào **một file** `App.js`. File đó sẽ dài khoảng 8.000 dòng. Muốn sửa cái nút "Thêm vào giỏ hàng" bạn phải cuộn chuột 10 phút để tìm.

Tách component giải quyết đúng ba nỗi đau:

| Nỗi đau | Nếu KHÔNG tách | Khi ĐÃ tách |
|---|---|---|
| **Lặp code** | Thẻ sản phẩm được copy 3 lần (trang chủ, thực đơn, sản phẩm liên quan) → 3 bản giống hệt | Viết 1 `ProductCard`, dùng ở cả 3 nơi |
| **Sửa một chỗ, sai mọi nơi** | Đổi màu nút phải nhớ sửa đủ 3 bản, quên 1 bản là giao diện lệch | Sửa 1 file, mọi nơi tự đổi theo |
| **Đọc không nổi** | Một hàm `return` dài 600 dòng JSX | `return` chỉ còn 10 dòng, mỗi dòng là một cái tên có nghĩa |

Nhìn vào `HomePage` của dự án — đây là ví dụ đẹp nhất về "tách xong thì trang chủ chỉ còn 7 dòng":

`yotea-fe/src/pages/user/HomePage.js:16-26`

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

Đọc đoạn này bạn hiểu ngay trang chủ có 7 khối, xếp theo thứ tự nào — mà **không cần biết** bên trong mỗi khối có bao nhiêu dòng Tailwind.

> ⚠️ **Nhưng dự án này tách chưa tới nơi:** hàm `renderStar` (vẽ 5 ngôi sao đánh giá) bị **copy-paste y hệt 4 lần** ở `ProductContent.js:105`, `ProductRelated.js:31`, `HomeProducts.js:67`, `CommentList.js:78`. Hàm `handleFavorites` bị copy 2 lần (`ProductContent.js:76-102` và `HomeProducts.js:38-64`). Đúng ra phải tách thành `<RatingStars value={...} />` và một custom hook `useFavorites()`. Đây là bài tập cuối bài.

---

## 2. Ba layout của dự án

**Layout** là cái khung bao quanh nội dung: phần *cố định* (header, menu, footer, sidebar) nằm ở layout; phần *thay đổi theo route* được React Router nhét vào chỗ `<Outlet />`.

```
┌─ WebsiteLayout ────────────────────┐
│  header (topbar + menu + search)   │  ← cố định
│  ┌──────────────────────────────┐  │
│  │  <Outlet />                  │  │  ← đổi theo URL
│  │  = HomePage / ProductPage /… │  │
│  └──────────────────────────────┘  │
│  footer                            │  ← cố định
└────────────────────────────────────┘
```

### 2.1. `WebsiteLayout` — khung cho toàn bộ trang khách

File dài **586 dòng**, nhưng bộ xương thì rất đơn giản.

`yotea-fe/src/pages/layouts/WebsiteLayout.js:39-53`

```jsx
const WebsiteLayout = () => {
  const dispatch = useDispatch();
  const [visible, setVisible] = useState(false);
  const [headerFixed, setHeaderFixed] = useState(false);
  const [productsSearch, setProductsSearch] = useState();
  const [isEmptyProduct, setIsEmptyProduct] = useState(false);
  const [keyword, setKeyword] = useState();
  const categories = useSelector(selectCatesProduct);

  const wishlist = useSelector(selectWishlist);
  const isShowWishlist = useSelector(selectShowWishlist);

  const isLogged = useSelector(selectStatusLoggin);
  const { user } = useSelector(selectAuth);
  const cart = useSelector(selectCart);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 41-42 | `visible`, `headerFixed` | Bật/tắt nút "lên đầu trang" và header dính khi cuộn quá 1000px |
| 43-45 | `productsSearch`, `isEmptyProduct`, `keyword` | Phục vụ ô tìm kiếm gợi ý trên header |
| 46 | `categories = useSelector(...)` | Danh mục sản phẩm để đổ vào menu xổ xuống |
| 48-49 | `wishlist`, `isShowWishlist` | Panel yêu thích trượt từ phải sang |
| 51-53 | `isLogged`, `user`, `cart` | Hiện tên người dùng và số món trong giỏ trên header |

Và đây là **trái tim** của layout — chỗ nội dung con được chèn vào:

`yotea-fe/src/pages/layouts/WebsiteLayout.js:377-382`

```jsx
        <section id="wishlist" className="wishlist" />
      </header>

      <main>
        <Outlet />
      </main>
```

Cấu trúc tổng thể: `<header>` (dòng 109 → 378) → `<main><Outlet /></main>` (380-382) → `<footer>` (384 → 523) → panel wishlist (525-581).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 377 có `<section id="wishlist" className="wishlist" />` **rỗng hoàn toàn** — đây là code chết sót lại từ bản HTML thuần, và nó **trùng `id="wishlist"`** với panel thật ở dòng 526. Trong HTML, một `id` chỉ được xuất hiện **đúng một lần**.

### 2.2. `AdminLayout` — sidebar + Outlet

`yotea-fe/src/pages/layouts/AdminLayout.js:17-24`

```jsx
const AdminLayout = () => {
  const { user } = useSelector(selectAuth);

  const dispatch = useDispatch();

  const handleLogout = () => {
    dispatch(logout());
  };
```

Layout này gọn hơn hẳn `WebsiteLayout`: **không có `useState`, không có `useEffect`** — nó chỉ cần biết `user` để hiện avatar, và một hàm đăng xuất.

Phần khung chính (đã lược các đoạn menu dài):

`yotea-fe/src/pages/layouts/AdminLayout.js:223-230`

```jsx
          </header>

          <main>
            <Outlet />
          </main>
        </div>
        <div className="fixed inset-0 z-10 w-screen h-screen bg-black bg-opacity-25 hidden dashboard__overlay" />
      </section>
```

Sidebar cố định bên trái (dòng 29-149) chứa 9 `NavLink`: `/admin/` (dòng 41), `/admin/cart` (56), `/admin/user` (66), `/admin/news` (78), `/admin/category-news` (90), `/admin/product` (102), `/admin/category` (114), `/admin/slider` (126), `/admin/contact` (138).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** ở dòng 209 có `<Link to="/admin/profile">Profile</Link>`, nhưng trong `App.js` **không hề có route `/admin/profile`**. Bấm vào là ra màn hình lỗi của router. Hai file `pages/admin/profile/AdminUpdateInfoPage.js` (257 dòng) và `AdminUpdatePassword.js` (169 dòng) đã được viết nhưng **chưa bao giờ được gắn route** → **code chết**, viết ra rồi bỏ đó.

### 2.3. `MyAccountLayout` — menu tài khoản + Outlet

File ngắn nhất trong ba layout, chỉ **80 dòng**.

`yotea-fe/src/pages/layouts/MyAccountLayout.js:6-14`

```jsx
const MyAccountLayout = () => {
  const dispatch = useDispatch();

  const { user } = useSelector(selectAuth);

  const handleLogout = () => {
    dispatch(logout());
    dispatch(clearWishlist());
  };
```

`yotea-fe/src/pages/layouts/MyAccountLayout.js:71-75`

```jsx
        </aside>
        <div className="col-span-12 lg:col-span-9">
          <Outlet />
        </div>
      </section>
```

Bên trái là `<aside>` (dòng 25-71) chứa avatar + 3 `NavLink` (`/my-account/`, `/my-account/update-password`, `/my-account/cart`) + nút Đăng xuất. Bên phải là `<Outlet />` chiếm 9/12 cột.

Layout này **lồng bên trong** `WebsiteLayout` — nghĩa là khi bạn vào `/my-account/cart`, React Router dựng ba tầng: `WebsiteLayout` → `MyAccountLayout` → `MyCartPage`. Đó chính là "layout lồng nhau" bạn học ở [Bài 15](15-routing-v6.md).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `<aside>` ở dòng 25 có class `hidden lg:block` → trên điện thoại **toàn bộ menu tài khoản biến mất**, và không có nút Đăng xuất nào khác thay thế. Người dùng mobile bị kẹt.

### 2.4. Bảng đối chiếu ba layout

| | `WebsiteLayout` | `AdminLayout` | `MyAccountLayout` |
|---|---|---|---|
| Số dòng | 586 | 235 | 80 |
| Khung cố định | header + menu + footer + panel wishlist | sidebar trái + header trên | menu tài khoản bên trái |
| Vị trí `<Outlet />` | dòng 381 | dòng 226 | dòng 73 |
| `useState` | 5 cái | không | không |
| `useEffect` | 1 cái (dòng 55-67) | không | không |
| Được bọc `PrivateRouter`? | ❌ công khai | ✅ `page="admin"` | ✅ `page="user"` |
| Đăng xuất có `clearWishlist`? | — | ❌ **quên** | ✅ có |

> ⚠️ Cột cuối là một bug thật: `AdminLayout.js:22-24` chỉ `dispatch(logout())` mà quên `clearWishlist()`. Kết quả: admin đăng xuất rồi, danh sách yêu thích của tài khoản cũ **vẫn còn trong Redux store**.

---

## 3. Props — cách rót dữ liệu xuống component

### 3.1. Truyền props và bóc tách ngay trên tham số

Component cha truyền dữ liệu xuống bằng cú pháp giống thuộc tính HTML:

`yotea-fe/src/pages/user/ProductPage.js:40-44`

```jsx
        <ProductContent
          getProducts={getAll}
          page={Number(page) || 1}
          url="thuc-don"
        />
```

Bên trong component con, ta **destructuring ngay trên tham số** để lấy từng prop ra biến riêng:

`yotea-fe/src/components/user/ProductContent.js:16`

```jsx
const ProductContent = ({ url, page, getProducts, parameter }) => {
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 41 | `getProducts={getAll}` | Truyền **một hàm** xuống. `ProductContent` không biết nó gọi API nào — cha quyết định |
| 42 | `page={Number(page) || 1}` | Ép chuỗi từ URL thành số; nếu không có thì mặc định trang 1 |
| 43 | `url="thuc-don"` | Chuỗi thuần thì dùng nháy kép, không cần `{ }` |
| 16 (con) | `({ url, page, getProducts, parameter })` | Bóc 4 prop ra 4 biến. `parameter` không được truyền ở đây → nhận `undefined` |

> 💡 **Vì sao truyền cả một hàm xuống?** Vì `ProductContent` được dùng lại ở **ba trang khác nhau**: `ProductPage` truyền `getAll`, `ProductByCate` truyền `getProductByCate`, `ProductSearchPage` truyền `search`. Cùng một giao diện lưới sản phẩm, ba nguồn dữ liệu. Đây là kỹ thuật **dependency injection** — rất đáng học.

Prop cũng có thể được **đổi tên khi bóc tách**, xem `CommentList`:

`yotea-fe/src/components/user/CommentList.js:12-17`

```jsx
const CommentList = ({
  productId,
  reRender: productDetailRerender,
  slug,
  page,
}) => {
```

Dòng 14 nghĩa là: *"nhận prop tên `reRender`, nhưng bên trong file này gọi nó là `productDetailRerender`"* — vì component còn có một state nội bộ **cũng tên** `reRender` (dòng 20), không đổi tên thì trùng biến.

### 3.2. `children` — prop đặc biệt "cái nằm giữa hai thẻ"

Khi bạn viết `<Cha><Con /></Cha>`, thì `<Con />` được React đóng gói thành prop tên **`children`** và truyền cho `Cha`.

`yotea-fe/src/components/admin/PrivateRouter.js:5-22`

```jsx
const PrivateRouter = ({ children, page }) => {
  const isLogged = useSelector(selectStatusLoggin);
  const auth = useSelector(selectAuth);

  if (page === "admin") {
    if (!isLogged) {
      return <Navigate to="/login" />;
    } else if (!auth.user.role || !auth.user.active) {
      return <Navigate to="/" />;
    }
  } else {
    if (!isLogged || !auth.user.active) {
      return <Navigate to="/login" />;
    }
  }

  return children;
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 5 | `({ children, page })` | `page` là prop thường; `children` là "hàng bên trong" |
| 9 | `if (page === "admin")` | Hai bộ luật khác nhau cho khu admin và khu người dùng |
| 11 | `return <Navigate to="/login" />` | Chưa đăng nhập → **đá** về trang đăng nhập, không render `children` |
| 13 | `return <Navigate to="/" />` | Đã đăng nhập nhưng không phải admin → về trang chủ |
| 21 | `return children` | Qua hết chốt → **trả lại đúng thứ được bọc bên trong** |

Và đây là chỗ nó được dùng — `children` chính là `<MyAccountLayout />`:

`yotea-fe/src/App.js:156-162`

```jsx
        {
          path: "my-account",
          element: (
            <PrivateRouter page="user">
              <MyAccountLayout />
            </PrivateRouter>
          ),
```

> 💡 **Cách nhớ:** prop thường viết **trong thẻ mở** (`page="user"`), còn `children` viết **giữa thẻ mở và thẻ đóng**. Một component "bao bọc" (wrapper) như `PrivateRouter`, `Modal`, `Card` gần như luôn dùng `children`.

> 🔒 **Ghi chú bảo mật:** `PrivateRouter` chỉ chặn **giao diện**, không chặn dữ liệu. Người dùng sửa `localStorage` là mở được UI admin. Tường thành thật nằm ở middleware `isAdmin` phía backend ([Bài 12](12-phan-quyen-middleware.md)). Chi tiết ở [Bài 24](24-private-router.md) và [Bài 33](33-ra-soat-bao-mat.md).

### 3.3. Props chảy một chiều: cha → con

React có quy tắc sắt: **dữ liệu chỉ chảy xuống**. Con muốn báo ngược lên cha thì cha phải truyền xuống **một hàm** để con gọi.

`yotea-fe/src/components/user/FilterProduct.js:4-10`

```jsx
const FilterProduct = ({
  filter,
  onUpdateFilter,
  start,
  limit,
  totalProduct,
}) => {
```

`filter` là dữ liệu đi xuống; `onUpdateFilter` là "cái chuông" để con rung lên báo cha. Quy ước đặt tên trong dự án: prop là hàm thì bắt đầu bằng **`on...`** (`onUpdateFilter`, `onSetTotal`, `onReRender`, `onHandleFavorites`).

---

## 4. Quy ước tổ chức: `pages/` vs `components/`

Cây thư mục `yotea-fe/src/`:

```
src/
├── App.js                    ← bảng route
├── api/                      ← tầng gọi API (Bài 18)
├── redux/                    ← store, slice (Bài 19-22)
├── utils/                    ← hàm dùng chung: formatCurrency, formatDate…
├── pages/
│   ├── layouts/              ← 3 layout: WebsiteLayout, AdminLayout, MyAccountLayout
│   ├── auth/                 ← LoginPage, RegisterPage, ForgotPage
│   ├── user/                 ← trang phía khách
│   └── admin/                ← trang phía quản trị
└── components/
    ├── Loading.js            ← dùng chung cả hai phía
    ├── user/                 ← component phía khách (20 file)
    └── admin/                ← component phía quản trị (13 file)
```

**Ranh giới giữa "page" và "component":**

| | **Page** (`pages/`) | **Component** (`components/`) |
|---|---|---|
| Gắn với route trong `App.js`? | ✅ Có | ❌ Không |
| Đọc `useParams()`? | ✅ Thường có | ❌ Hiếm |
| Gọi `updateTitle()` đổi tiêu đề tab? | ✅ Có | ❌ Không |
| Tự lấy dữ liệu? | Có, hoặc uỷ quyền cho con | Tuỳ — xem cảnh báo dưới |
| Nhận props? | Không (nhận qua URL) | ✅ Đó là cách sống của nó |
| Ví dụ | `ProductPage`, `CartPage` | `ProductCard`, `Pagination` |

Xem `ProductPage` để thấy đúng vai một "page":

`yotea-fe/src/pages/user/ProductPage.js:8-13`

```jsx
const ProductPage = () => {
  const { page } = useParams();

  useEffect(() => {
    updateTitle("Thực đơn");
  }, []);
```

Nó đọc `:page` từ URL, đặt tiêu đề tab, rồi giao toàn bộ việc hiển thị cho `<NavProduct />` và `<ProductContent />`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn (rất quan trọng):** lý thuyết chuẩn nói component nên là **"trình bày thuần"** — chỉ nhận props và vẽ. Nhưng trong Yotea, rất nhiều component **tự gọi API**: `ProductContent` gọi 20 request mỗi trang, `NavProduct` tự lấy danh mục, `HomeProducts` tự lấy 8 sản phẩm. Hệ quả là trang cha **không kiểm soát** được lúc nào dữ liệu về, không tái sử dụng được component ở nơi khác, và không test được. Cách làm chuẩn hơn: page (hoặc Redux) lo dữ liệu, component chỉ nhận `products` qua prop rồi vẽ. Ta sẽ làm đúng như vậy với `ToppingCard` ở mục 8.

> ⚠️ **File `App.css` không được dùng.** Thư mục `src/` có `App.css` nhưng grep toàn bộ source chỉ thấy `index.js:3` import `./index.css`. `App.css` là tàn dư của template `create-react-app`, chưa ai xoá. Toàn bộ style của dự án đến từ Tailwind ([Bài 17](17-tailwind-css.md)) cộng vài class thủ công trong `index.css`.

---

## 5. Điểm danh các component tái sử dụng đáng chú ý

### 5.1. Dùng chung cả hai phía

| Component | Props | Một câu mô tả |
|---|---|---|
| `components/Loading.js` | `{ active }` | Lớp phủ đen mờ + vòng xoay `PuffLoader`, bật/tắt bằng prop `active`. |
| `components/admin/PrivateRouter.js` | `{ children, page }` | Cổng gác route: chưa đăng nhập / sai quyền thì `<Navigate>` đi chỗ khác. |

`yotea-fe/src/components/Loading.js:3-9`

```jsx
const Loading = ({ active }) => {
  return (
    <div
      id="loading"
      className={`${
        active && "active"
      } z-20 invisible fixed top-0 right-0 bottom-0 left-0`}
```

### 5.2. Phía khách — `components/user/`

| Component | Props | Một câu mô tả |
|---|---|---|
| `Pagination` | `{ page, totalPage, url }` | Dãy nút tròn màu `#D9A953`, sinh link `/${url}/page/${i}`. |
| `ProductContent` | `{ url, page, getProducts, parameter }` | Lưới/danh sách sản phẩm + bộ lọc + phân trang; nhận hàm API từ cha. |
| `FilterProduct` | `{ filter, onUpdateFilter, start, limit, totalProduct }` | Thanh chọn kiểu xem lưới/danh sách và tiêu chí sắp xếp. |
| `NavProduct` | `{ cateId }` | Sidebar trang thực đơn: danh mục + top 10 sản phẩm được yêu thích. |
| `NavNews` | `{ slug }` | Thanh chuyên mục tin tức nằm ngang, tô đậm mục đang xem. |
| `NewsContent` | `{ page, getNews, parameter, url }` | Bản sao của `ProductContent` nhưng cho bài viết, `limit = 8`. |
| `CartNav` | `{ page }` | Breadcrumb 3 bước: SHOPPING CART → Checkout details → Order Complete. |
| `CommentList` | `{ productId, reRender, slug, page }` | Danh sách bình luận đã ghép với điểm sao, có phân trang `limit = 4`. |
| `CommentProduct` | `{ productId, onReRender, productData }` | Form chấm sao + viết bình luận. |
| `ProductRelated` | `{ id, cateId, onHandleFavorites }` | 4 sản phẩm cùng danh mục ở cuối trang chi tiết. |
| `SidebarNews` | `{ cateId, newsId }` | Cột phải trang bài viết: chuyên mục + 10 bài mới nhất. |
| `Iframe` | `{ iframe }` | Đổ chuỗi HTML bản đồ Google từ DB bằng `dangerouslySetInnerHTML`. |
| `home/HomeBanner` | không | Slider ảnh đầu trang chủ (react-slick). |
| `home/HomeCategory` | không | Lưới 4 ô danh mục sản phẩm. |
| `home/HomeProducts` | không | 8 sản phẩm nổi bật kèm số sao. |
| `home/HomeNews` | không | 4 bài viết mới nhất. |
| `home/HomeWhy` · `home/HomeFeedback` · `home/HomeShow` | không | Ba khối **nội dung tĩnh 100%**: "tại sao chọn chúng tôi", 3 feedback bịa, 8 ảnh "Instagram". |

### 5.3. Phía quản trị — `components/admin/`

| Component | Props | Một câu mô tả |
|---|---|---|
| `AdminPagination` | `{ page, totalPage, url }` | Y hệt `Pagination` về thuật toán nhưng khác skin, tự chèn tiền tố `/admin`. |
| `ListProduct` | `{ start, limit }` | Bảng sản phẩm dùng RTK Query, có nút Sửa/Xoá. |
| `NewsList` · `UserList` · `ContactList` · `ListSlider` · `CateNewsList` | `{ start, limit }` (trừ 2 cái cuối) | Bảng dữ liệu tương ứng, cùng một khuôn. |
| `OrderList` | `{ onSetTotal, start, limit }` | Bảng đơn hàng, báo tổng số về cho trang cha qua `onSetTotal`. |
| `NextArrow` · `PrevArrow` | `{ onClick }` | Hai nút mũi tên cho react-slick. |

Hãy so sánh hai component phân trang — cùng props, khác giao diện:

`yotea-fe/src/components/user/Pagination.js:5`

```jsx
const Pagination = ({ page, totalPage, url }) => {
```

`yotea-fe/src/components/admin/AdminPagination.js:3`

```jsx
const AdminPagination = ({ page, totalPage, url }) => {
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:**
> 1. `components/admin/StoreList.js` (126 dòng) và `components/admin/AdminCommentList.js` (131 dòng) **không được import ở bất kỳ đâu** → code chết.
> 2. `PrivateRouter`, `NextArrow`, `PrevArrow` nằm trong `components/admin/` nhưng lại **được dùng bởi phía khách** (`HomeBanner`, `HomeFeedback`, `HomeShow` import chéo `../../admin/NextArrow`). Đặt sai chỗ.
> 3. `Pagination` và `AdminPagination` **trùng thuật toán 100%** — chỉ nên có một component nhận thêm prop `variant`.

---

## 6. `key` trong `.map()` — thứ ai cũng bỏ qua và ai cũng dính lỗi

### 6.1. React cần `key` để làm gì?

Khi danh sách thay đổi, React so sánh cây cũ với cây mới để biết **phải sửa những nút DOM nào**. Không có `key`, nó phải so từng phần tử theo **vị trí**. Có `key`, nó so theo **danh tính**.

Ví dụ đời thường: lớp học có 30 học sinh. Nếu điểm danh theo **số thứ tự chỗ ngồi** (index), một bạn nghỉ học là toàn bộ số thứ tự phía sau dồn lên → cô giáo tưởng cả lớp đổi người. Nếu điểm danh theo **mã học sinh** (`_id`), ai vắng thì đúng người đó vắng.

### 6.2. Dự án dùng `key={index}` ở khắp nơi

`yotea-fe/src/pages/layouts/WebsiteLayout.js:252-261`

```jsx
                    {categories?.map((item, index) => (
                      <li key={index}>
                        <Link
                          to={`/danh-muc/${item.slug}`}
                          className="block py-1.5 text-gray-500 transition ease-linear duration-200 hover:text-[#D9A953]"
                        >
                          {item.name}
                        </Link>
                      </li>
                    ))}
```

`yotea-fe/src/components/user/ProductContent.js:199-200`

```jsx
              {products?.map((item, index) => (
                <div className="group" key={index}>
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** grep toàn bộ `yotea-fe/src` cho ra **39 lần dùng `key={index}` trên 34 file** — nghĩa là gần như **mọi** danh sách trong dự án. Đúng ra phải là `key={item._id}`, vì MongoDB đã cấp sẵn `_id` duy nhất cho từng bản ghi.

### 6.3. Hậu quả cụ thể của `key={index}`

| Tình huống | Với `key={index}` | Với `key={item._id}` |
|---|---|---|
| Xoá phần tử **giữa** danh sách | React tưởng phần tử cuối bị xoá → các phần tử sau bị vẽ lại sai | Xoá đúng nút DOM tương ứng |
| Người dùng đang gõ vào ô input trong dòng 2, rồi dòng 1 bị xoá | **Chữ đang gõ nhảy sang dòng khác** | Chữ ở nguyên chỗ |
| Sắp xếp lại (đổi sort giá tăng/giảm) | React vẽ lại toàn bộ, ảnh nhấp nháy | React chỉ **di chuyển** nút DOM, mượt |
| Hiệu năng | Kém khi danh sách dài | Tốt |

Trong Yotea, hậu quả rõ nhất nằm ở trang **thực đơn**: đổi tiêu chí sắp xếp (`ProductContent` gọi lại API, mảng `products` thay đổi hoàn toàn) nhưng `key` vẫn là `0,1,2,…,8` → React giữ nguyên toàn bộ nút DOM cũ và ghi đè nội dung, khiến ảnh sản phẩm bị "chớp" khi tải lại.

> 💡 **Quy tắc vàng:** `key` phải **ổn định** (không đổi giữa các lần render), **duy nhất** trong cùng một danh sách, và **thuộc về chính dữ liệu** (không phải vị trí). Chỉ được dùng `index` khi danh sách **không bao giờ** thêm/xoá/sắp xếp lại — ví dụ mảng 5 mức đá `[0, 30, 50, 70, 100]`.

---

## 7. Hiển thị có điều kiện trong JSX

### 7.1. Toán tử `&&` — "có thì hiện, không thì thôi"

`yotea-fe/src/pages/user/cart/CheckoutPage.js:206-220`

```jsx
                {user && (
                  <div className="col-span-12 mb-3 flex items-center">
                    <input
                      type="checkbox"
                      id="cart__checkout-save-address"
                      {...register("saveAddress")}
                    />
                    <label
                      htmlFor="cart__checkout-save-address"
                      className="ml-1 block text-md"
                    >
                      Lưu thông tin thanh toán?
                    </label>
                  </div>
                )}
```

Đọc: *"chỉ vẽ ô 'Lưu thông tin thanh toán' khi đã đăng nhập."* Khách vãng lai sẽ không thấy nó.

Cùng kiểu, `Pagination` chỉ vẽ dãy số khi có nhiều hơn 1 trang:

`yotea-fe/src/components/user/Pagination.js:39`

```jsx
      {totalPage > 1 && pagination}
```

### 7.2. Toán tử ba ngôi — "cái này hoặc cái kia"

`yotea-fe/src/components/user/ProductContent.js:128-133`

```jsx
      {emptyProduct ? (
        <div className="col-span-12 lg:col-span-9">
          Không tìm thấy sản phẩm nào
        </div>
      ) : (
        <div className="col-span-12 lg:col-span-9">
```

`yotea-fe/src/pages/user/cart/CartPage.js:131`

```jsx
        {cart.length ? (
```

### 7.3. ⚠️ Bẫy chết người: giá trị `0` bị in ra màn hình

Đây là bẫy kinh điển nhất của React. Xem `CartPage.js:131` ở trên — nếu tác giả viết bằng `&&` thay vì ba ngôi:

```jsx
// ❌ CÁCH SAI — đoạn này bạn tự thử, dự án KHÔNG viết như vậy
{cart.length && (
  <table>...</table>
)}
```

Khi giỏ hàng rỗng, `cart.length` là `0`. Toán tử `&&` trả về **`0`** (chứ không phải `false`), mà React thì **có in số `0` ra màn hình**. Kết quả: giữa trang giỏ hàng trống hiện lên một con số `0` lơ lửng, không ai hiểu ở đâu ra.

**Bảng tra nhanh — giá trị nào bị in ra?**

| Biểu thức | `&&` trả về | React vẽ gì? |
|---|---|---|
| `false && <div/>` | `false` | Không vẽ gì ✅ |
| `undefined && <div/>` | `undefined` | Không vẽ gì ✅ |
| `null && <div/>` | `null` | Không vẽ gì ✅ |
| `0 && <div/>` | **`0`** | **In ra chữ `0`** ❌ |
| `"" && <div/>` | `""` | Không vẽ gì ✅ |
| `NaN && <div/>` | **`NaN`** | **In ra chữ `NaN`** ❌ |

**Ba cách chữa:**

```jsx
{cart.length > 0 && <table>...</table>}      // ép thành boolean bằng phép so sánh
{!!cart.length && <table>...</table>}        // ép bằng hai dấu !
{cart.length ? <table>...</table> : null}    // dùng ba ngôi — cách dự án đang dùng
```

### 7.4. ⚠️ Bẫy thứ hai: `&&` trong chuỗi `className`

`yotea-fe/src/components/user/CartNav.js:12-14`

```jsx
            className={`${
              page === "list" && "text-black"
            } uppercase text-gray-400 transition ease-linear duration-200 hover:text-black`}
```

`yotea-fe/src/components/user/NavNews.js:22-24`

```jsx
        className={`text-center px-4 group flex flex-col items-center cate-news-item ${
          !slug && "active"
        }`}
```

Khi điều kiện **sai**, biểu thức trả về `false`, và template literal biến `false` thành **chuỗi `"false"`**. DOM thật sẽ là:

```html
<a class="false uppercase text-gray-400 …">SHOPPING CART</a>
```

Tailwind không có class tên `false` nên **không vỡ giao diện**, nhưng HTML bị bẩn và rất khó debug. Cách viết đúng:

```jsx
className={`${page === "list" ? "text-black" : ""} uppercase text-gray-400 …`}
```

Lỗi này lặp ở `CartNav.js:13,26,38`, `NavNews.js:24,40`, `SidebarNews.js:36,55` và `Loading.js:8`.

---

## 8. 🛠️ Tự tay làm — `ToppingCard` và `ToppingPage`

> Mục tiêu phần này: cuối phần bạn mở `http://localhost:3000/topping` và thấy **một lưới 6 thẻ topping** có ảnh, tên và giá tiền định dạng `5.000 ₫` — dựng từ đúng một component tái sử dụng.

Ở bài trước bạn đã có route `topping`. Bây giờ ta đổ nội dung thật vào nó. **Ở bài này ta cố tình dùng dữ liệu giả cứng trong mảng** — để tập trung 100% vào props, `.map()` và `key`. Việc gọi API `/api/toppings` thật sẽ làm ở [Bài 18](18-tang-api-axios.md) và [Bài 20](20-async-thunk.md).

### Bước 1 — Tạo component `ToppingCard`

Tạo **file mới** `yotea-fe/src/components/user/ToppingCard.js`. Đặt trong `components/user/` vì đây là component phía khách, và nó **không tự lấy dữ liệu** — nó chỉ nhận props và vẽ.

```jsx
// yotea-fe/src/components/user/ToppingCard.js  ← file MỚI, bạn tự tạo (dự án chưa có)
import { formatCurrency } from "../../utils";

const ToppingCard = ({ topping }) => {
  return (
    <div className="group">
      <div className="relative bg-[#f7f7f7] overflow-hidden">
        <div
          style={{ backgroundImage: `url(${topping.image})` }}
          className="bg-cover pt-[100%] bg-center block"
        />
      </div>
      <div className="text-center py-3">
        <h3 className="block font-semibold text-lg">{topping.name}</h3>
        <div className="text-sm pt-1">{formatCurrency(topping.price)}</div>
      </div>
    </div>
  );
};

export default ToppingCard;
```

**Giải thích từng ý:**

| Dòng | Ý nghĩa |
|---|---|
| `import { formatCurrency }` | Lấy hàm định dạng tiền có sẵn của dự án (`yotea-fe/src/utils/index.js:14-15`) — **không tự viết lại** |
| `({ topping })` | Nhận **đúng một prop** tên `topping`, bóc tách ngay trên tham số |
| `style={{ backgroundImage: ... }}` | Hai lớp ngoặc nhọn: lớp ngoài là "đây là JS", lớp trong là object CSS |
| `pt-[100%]` | Thủ thuật của dự án: padding-top 100% tạo ô vuông tỉ lệ 1:1 |
| `formatCurrency(topping.price)` | `35000` → `35.000 ₫` |

Khung Tailwind ở trên **bắt chước đúng thẻ sản phẩm sẵn có** của dự án:

`yotea-fe/src/components/user/ProductContent.js:217-233`

```jsx
                  <div className="text-center py-3">
                    <p className="uppercase text-xs text-gray-400">
                      {item.categoryId?.name}
                    </p>
                    <Link
                      to={`/san-pham/${item.slug}`}
                      className="block font-semibold text-lg"
                    >
                      {item.name}
                    </Link>
                    <ul className="flex text-yellow-500 text-xs justify-center pt-1">
                      {renderStar(item.ratingNumber || 0)}
                    </ul>
                    <div className="text-sm pt-1">
                      {formatCurrency(item.price)}
                    </div>
                  </div>
```

### Bước 2 — Viết lại `ToppingPage` để dùng `ToppingCard`

Mở file `yotea-fe/src/pages/user/ToppingPage.js` (bạn đã tạo ở Bài 15) và **thay toàn bộ nội dung**:

```jsx
// yotea-fe/src/pages/user/ToppingPage.js  ← file bạn đã tạo ở Bài 15, giờ viết lại
import { useEffect } from "react";
import ToppingCard from "../../components/user/ToppingCard";
import { updateTitle } from "../../utils";

// ⚠️ Dữ liệu GIẢ, cứng trong file — Bài 18/20 sẽ thay bằng dữ liệu thật từ API
const ANH_TAM =
  "https://res.cloudinary.com/levantuan/image/upload/v1644302455/assignment-js/thumbnail-image-vector-graphic-vector-id1147544807_ochvyr.jpg";

const TOPPINGS_GIA = [
  { _id: "tp1", name: "Trân châu đen", price: 5000, image: ANH_TAM },
  { _id: "tp2", name: "Trân châu trắng", price: 6000, image: ANH_TAM },
  { _id: "tp3", name: "Thạch phô mai", price: 10000, image: ANH_TAM },
  { _id: "tp4", name: "Pudding trứng", price: 8000, image: ANH_TAM },
  { _id: "tp5", name: "Thạch dừa", price: 5000, image: ANH_TAM },
  { _id: "tp6", name: "Kem cheese", price: 12000, image: ANH_TAM },
];

const ToppingPage = () => {
  useEffect(() => {
    updateTitle("Topping");
  }, []);

  return (
    <section className="container max-w-6xl mx-auto px-3 my-8">
      <h1 className="text-[#D9A953] font-semibold text-3xl mb-5 text-center">
        Danh sách topping
      </h1>

      <div className="grid grid-cols-2 md:grid-cols-3 gap-3">
        {TOPPINGS_GIA.map((topping) => (
          <ToppingCard key={topping._id} topping={topping} />
        ))}
      </div>
    </section>
  );
};

export default ToppingPage;
```

**Ba điểm cần soi kỹ:**

1. **`key={topping._id}`** — không dùng `index`. Mỗi topping có mã riêng, đó mới là danh tính thật.
2. **`topping={topping}`** — truyền cả object xuống. Bên `ToppingCard` bóc ra bằng `({ topping })`.
3. **`ToppingPage` là "page"**: nó có route, nó gọi `updateTitle`, nó nắm dữ liệu. **`ToppingCard` là "component"**: không biết dữ liệu ở đâu ra, chỉ nhận và vẽ. Đây chính là ranh giới ở mục 4.

> 💡 **Vì sao viết `TOPPINGS_GIA` ở NGOÀI component?** Vì nếu để bên trong, mảng sẽ được **tạo lại mỗi lần render** — vô ích, và sau này nếu bạn đưa nó vào `useEffect` deps thì effect sẽ chạy vô hạn. Hằng số không phụ thuộc state thì luôn đặt ngoài.

### Bước 3 — Kiểm tra file `App.js` (không sửa gì thêm)

Route bạn thêm ở Bài 15 đã trỏ sẵn tới `ToppingPage`. Chỉ cần chắc rằng nó nằm **trong mảng `children` của `WebsiteLayout`** thì trang mới có header/footer bao quanh.

---

## 9. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-fe
npm start
```

Mở `http://localhost:3000/topping`. Bạn **phải** thấy đủ 5 điều sau:

| # | Phải nhìn thấy |
|---|---|
| 1 | Header vàng của Yotea ở trên, footer ảnh nền ở dưới → chứng tỏ trang nằm trong `WebsiteLayout` |
| 2 | Tiêu đề vàng **"Danh sách topping"** |
| 3 | **6 thẻ** xếp lưới: 2 cột trên điện thoại, 3 cột trên máy tính |
| 4 | Giá hiển thị dạng **`5.000 ₫`**, `10.000 ₫`… (dấu chấm ngăn nghìn, ký hiệu ₫ ở cuối) |
| 5 | Tab trình duyệt ghi **"Topping - Trà sữa Yotea"** |

**Kiểm chứng `key` bằng React DevTools:**

1. Cài extension **React Developer Tools**, mở tab ⚛️ Components.
2. Tìm `ToppingPage` → mở ra sẽ thấy 6 `ToppingCard`.
3. Bấm vào một `ToppingCard` bất kỳ → panel bên phải hiện `props: { topping: {…} }`.
4. Mở tab **Console** → **không được có** dòng cảnh báo đỏ `Warning: Each child in a list should have a unique "key" prop`.

**Thử nghiệm hiểu bài (làm rồi hoàn tác):** tạm xoá `key={topping._id}` đi và F5. Console sẽ hiện đúng cảnh báo trên. Đó là cách React nhắc bạn.

---

## 10. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Warning: Each child in a list should have a unique "key" prop` | Quên `key` trong `.map()` | Thêm `key={item._id}` vào **phần tử ngoài cùng** của mỗi vòng lặp |
| `Encountered two children with the same key` | Hai phần tử trùng `_id`, hoặc trộn hai danh sách vào chung | Kiểm tra dữ liệu; nếu ghép nhiều nguồn thì thêm tiền tố: `` key={`tp-${item._id}`} `` |
| `Cannot read properties of undefined (reading 'price')` | Truyền sai tên prop: cha viết `data={...}` nhưng con nhận `({ topping })` | Đối chiếu tên prop ở **cả hai** file, phải trùng khít |
| `topping.price.toLocaleString is not a function` | `price` là chuỗi `"5000"` chứ không phải số | `formatCurrency` chỉ nhận số → dùng `formatCurrency(+topping.price)` |
| `Module not found: Can't resolve '../../utils'` | Sai số cấp `../` | Từ `components/user/` về `src/` là **2 cấp**; từ `pages/user/cart/` là **3 cấp** |
| Trang trắng, console báo `Objects are not valid as a React child` | Lỡ viết `{topping}` thay vì `{topping.name}` | React vẽ được chuỗi/số, **không vẽ được object** |
| Số `0` lạ hiện giữa trang | Dùng `{mang.length && <div/>}` với mảng rỗng | Đổi sang `{mang.length > 0 && ...}` hoặc ba ngôi (mục 7.3) |
| Trang hiện nhưng **không có header/footer** | Route `topping` đặt ngoài `children` của `WebsiteLayout` | Xem lại `App.js`, [Bài 15](15-routing-v6.md) |

---

## 11. 📝 Bài tập

**Bài 1.** Trong `ToppingCard`, hãy thêm một prop mới `isHot` (kiểu boolean). Khi `isHot` đúng thì hiện thêm nhãn đỏ **"HOT"** ở góc trên bên phải ảnh; khi sai thì không hiện gì. Đánh dấu `Thạch phô mai` và `Kem cheese` là hot.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thêm `isHot` vào dữ liệu (file `ToppingPage.js`):

```jsx
// code bạn tự viết thêm
{ _id: "tp3", name: "Thạch phô mai", price: 10000, image: ANH_TAM, isHot: true },
{ _id: "tp6", name: "Kem cheese",   price: 12000, image: ANH_TAM, isHot: true },
```

Truyền xuống:

```jsx
<ToppingCard key={topping._id} topping={topping} isHot={topping.isHot} />
```

Trong `ToppingCard.js`:

```jsx
const ToppingCard = ({ topping, isHot }) => {
  return (
    <div className="group">
      <div className="relative bg-[#f7f7f7] overflow-hidden">
        {isHot && (
          <span className="absolute top-2 right-2 z-10 bg-red-600 text-white text-xs px-2 py-0.5 rounded">
            HOT
          </span>
        )}
        <div
          style={{ backgroundImage: `url(${topping.image})` }}
          className="bg-cover pt-[100%] bg-center block"
        />
      </div>
      ...
```

**Vì sao dùng `&&` ở đây an toàn?** Vì `isHot` là boolean thật (`true`/`undefined`), không phải số — không dính bẫy `0` ở mục 7.3. Nếu bạn đổi sang `{topping.soLuongDaBan && <span>…</span>}` thì lại dính ngay khi số lượng bằng 0.

Lưu ý: `<span>` dùng `absolute` nên thẻ cha bắt buộc phải có `relative` — may là dự án đã có sẵn `className="relative bg-[#f7f7f7] overflow-hidden"`.

</details>

**Bài 2.** Hiện tại `ToppingPage` tự chứa mảng dữ liệu **và** tự vẽ lưới. Hãy tách phần vẽ lưới ra thành component mới `src/components/user/ToppingList.js` nhận prop `{ toppings }`, để `ToppingPage` chỉ còn nắm dữ liệu. Sau đó giải thích: bạn vừa dịch chuyển ranh giới "page / component" theo hướng nào?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```jsx
// yotea-fe/src/components/user/ToppingList.js  ← file MỚI, bạn tự tạo
import ToppingCard from "./ToppingCard";

const ToppingList = ({ toppings }) => {
  if (!toppings?.length) {
    return <div className="text-center py-8">Chưa có topping nào</div>;
  }

  return (
    <div className="grid grid-cols-2 md:grid-cols-3 gap-3">
      {toppings.map((topping) => (
        <ToppingCard key={topping._id} topping={topping} />
      ))}
    </div>
  );
};

export default ToppingList;
```

`ToppingPage` gọn lại còn:

```jsx
      <ToppingList toppings={TOPPINGS_GIA} />
```

**Trả lời câu hỏi:** bạn đã đẩy `ToppingPage` về đúng vai **"page thuần dữ liệu"** — nó chỉ lo *có gì*, không lo *vẽ thế nào*. Và `ToppingList` là **component trình bày thuần**: đưa mảng nào nó vẽ mảng đó, dùng lại được ở trang chi tiết sản phẩm, ở trang chủ, ở đâu cũng được.

Để ý thêm: `ToppingList` còn xử lý luôn **trạng thái rỗng** — thứ mà `ProductContent.js:128` của dự án cũng làm (`emptyProduct ? … : …`). Đây là thói quen tốt: mọi danh sách đều phải trả lời được câu hỏi "nếu không có gì thì hiện gì?".

</details>

**Bài 3.** Đọc `yotea-fe/src/components/user/CartNav.js:9-18` rồi trả lời: `CartNav` nhận prop gì, prop đó có bao nhiêu giá trị hợp lệ, và dòng 13 sai ở chỗ nào? Viết lại dòng 12-14 cho đúng.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

- **Prop**: đúng một prop `page` (`CartNav.js:5`).
- **Giá trị hợp lệ**: ba — `"list"` (trang `/cart`), `"checkout"` (trang `/checkout`), `"thank-you"` (trang `/thank-you`). Prop này chỉ để **tô đậm bước hiện tại** trong breadcrumb 3 bước.
- **Chỗ sai ở dòng 13**: `page === "list" && "text-black"` nằm bên trong template literal. Khi điều kiện sai, `&&` trả về `false`, và template literal ép `false` thành **chuỗi `"false"`** → DOM có `class="false uppercase text-gray-400 …"`. Không vỡ giao diện (Tailwind bỏ qua class lạ) nhưng là rác.

Viết lại:

```jsx
            className={`${
              page === "list" ? "text-black" : ""
            } uppercase text-gray-400 transition ease-linear duration-200 hover:text-black`}
```

Lỗi này còn ở `CartNav.js:26` và `:38`, `NavNews.js:24,40`, `SidebarNews.js:36,55`, `Loading.js:8`.

</details>

**Bài 4.** (Nâng cao) Hàm `renderStar` bị copy-paste 4 lần trong dự án. Hãy thiết kế — **chỉ viết ra file mới, TUYỆT ĐỐI không sửa 4 file cũ** — một component `src/components/user/RatingStars.js` nhận prop `{ value }` (số sao vàng, 0–5) và vẽ đủ 5 ngôi sao.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```jsx
// yotea-fe/src/components/user/RatingStars.js  ← file MỚI, bạn tự tạo
import { faStar } from "@fortawesome/free-solid-svg-icons";
import { FontAwesomeIcon } from "@fortawesome/react-fontawesome";

const RatingStars = ({ value = 0 }) => {
  const total = 5;
  const filled = Math.min(Math.max(Math.round(value), 0), total);

  return (
    <ul className="flex text-yellow-500 text-xs justify-center pt-1">
      {Array.from({ length: total }, (_, i) => (
        <li key={i} className={i < filled ? "text-yellow-500" : "text-gray-300"}>
          <FontAwesomeIcon icon={faStar} />
        </li>
      ))}
    </ul>
  );
};

export default RatingStars;
```

**Vì sao ở đây `key={i}` lại CHẤP NHẬN ĐƯỢC?** Vì danh sách này **luôn đúng 5 phần tử**, không bao giờ thêm/xoá/sắp xếp lại. Đây đúng là ngoại lệ đã nói ở mục 6.3. Ngược lại, danh sách sản phẩm thì tuyệt đối không được.

Ba cải tiến so với bản gốc của dự án:
1. **Kẹp giá trị** bằng `Math.min/Math.max` — bản gốc không kẹp, nên `ratingNumber = 7` sẽ vẽ ra 7 sao vàng và −2 sao xám.
2. **Làm tròn** bằng `Math.round` — bản gốc dựa vào việc `getAvgStar` đã `Math.ceil` sẵn; nếu API đổi, vòng `for` sẽ chạy số thập phân.
3. **Một vòng lặp** thay vì hai vòng đẩy vào mảng.

Dùng nó ở `ToppingCard` (nếu topping có điểm sao):

```jsx
<RatingStars value={topping.rating} />
```

</details>

---

## 📌 Tóm tắt

- Tách component để **không lặp code, sửa một chỗ áp dụng mọi nơi, và đọc được**. `HomePage.js:16-26` là ví dụ mẫu: 7 dòng mô tả cả trang chủ.
- Yotea có **ba layout**: `WebsiteLayout` (header + menu + footer, `<Outlet />` ở dòng 381), `AdminLayout` (sidebar, `<Outlet />` ở dòng 226), `MyAccountLayout` (menu tài khoản, `<Outlet />` ở dòng 73).
- **Props** truyền cha → con, bóc tách ngay trên tham số: `({ url, page, getProducts })`. Prop có thể là **chuỗi, số, object, mảng, và cả hàm**.
- **`children`** là prop đặc biệt chứa "phần nằm giữa hai thẻ" — `PrivateRouter` sống nhờ nó.
- **Page** = có route, đọc `useParams`, đặt tiêu đề, nắm dữ liệu. **Component** = nhận props, vẽ. Dự án này phá vỡ ranh giới đó ở nhiều chỗ (component tự gọi API) — biết để tránh.
- **`key` phải là `_id`, không phải `index`.** Dự án dùng `key={index}` **39 lần trên 34 file** — kinh điển sai.
- Trong JSX: `&&` để "có thì hiện", ba ngôi để "cái này hoặc cái kia". **Bẫy: `0 && <div/>` in ra số `0`**, và `${cond && "class"}` in ra chuỗi `"false"`.
- Code chết cần biết: 2 trang `pages/admin/profile/*` không có route, 2 component `StoreList`/`AdminCommentList` không được import, và file `src/App.css` **không được import ở đâu cả**.

**Từ khoá tra cứu thêm:** `react props`, `react children prop`, `react key prop index anti-pattern`, `react conditional rendering`, `react outlet layout route`, `presentational vs container components`

➡️ **Bài tiếp theo:** [17 — Tailwind CSS trong dự án](17-tailwind-css.md) — thẻ `ToppingCard` của bạn hiện còn khá thô; bài sau ta sẽ mổ xẻ đống class Tailwind của Yotea và trang trí nó cho ra dáng một thẻ sản phẩm thật.
